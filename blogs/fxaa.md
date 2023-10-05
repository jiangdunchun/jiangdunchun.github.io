# FXAA - 我给Piccolo贡献了第一个feature

---

<center><img style="max-width: 80%;" src="blogs/fxaa/fxaa_result.png"></center>

在GAMES105课程的第一次作业要求公布时，我就意识到FXAA完美的符合了附加题的要求。我于是找到了自己之前在[shadertoy上实现的FXAA](https://www.shadertoy.com/view/stlSzf)，并将其塞入到Piccolo的渲染流程中。虽然reviewer也曾怀疑这个feature是否会影响渲染效率，并增加了Piccolo的复杂性，但最后感谢他们还是同意了合并请求。感谢王希老师和社区的小伙伴，构建了这样一个有趣的社区，并让我有幸成为王希老师口中的"第一个完成贡献feature的小伙伴"。

<center><img style="max-width: 80%;" src="blogs/fxaa/wall.PNG"></center>

<center><p>图0 上墙瞬间(也就来回看了几十遍)</p></center>

## 添加FXAA的render pass

<center><img style="max-width: 80%;" src="blogs/fxaa/insert_fxaa_pass.png"></center>

<center><p>图1 添加FXAA的render pass</p></center>

在接触Piccolo之前，我更多的使用OpenGL去写一些渲染效果。因此，当我看到Vulkan如此繁琐的开发模式后，我一度想放弃，但最后还是战胜了恐惧。我发挥了程序员的必备技能:既然Color Grading也是一个后处理pass, ctrl+c和ctrl+v后做一点点修改，不就是FXAA的pass了吗?那么，重点就是需要修改哪些部分?

首先分析一下原有流程。Color Grading pass将输出一张纹理，作为ImGUI pass的输入。那么，新加入的FXAA将以Color Grading输出的纹理作为输入，输出一张新的纹理作为ImGUI pass的输入。所以，流程如下:

> 1. 类似于ColorGrading的shader，新建FXAA的vertex shader和fragment shader。其中vertex shader与ColorGrading的完全一致，fragment shader则先直接简单的输出color。
> 2. 类似于ColorGradingPass，新建一个FXAAPass，绑定新建的shader。
> 3. 在创建和设置ColorGradingPass的地方创建和设置FXAAPass。这时候需要添加新的frame buffer，并小心修改pipeline的state。

这里最麻烦的是第3步，很多细节需要处理。但即使对Vulkan不熟悉，参考ColorGradingPass的相关代码，也大概可以完成。我原本希望在FXAAPass直接使用已有的frame buffer，但因为FXAA的独特性(需要线性采样等），修改了原来frame buffer的一些属性。在reviewer的建议下，我新建了frame buffer来处理与FXAA类似的后处理pass。

## 算法详解

在了解FXAA的fragment shader之前，我们先来简单的了解一下FXAA的原理。大部分的锯齿都出现在物体边缘或者高光变化的部分，我们可以通过对已有渲染结果进行后处理的方式，检测出像素之间的边缘，然后根据边缘信息对边缘两侧的像素进行混合，达到抗锯齿的效果。

<center><img style="max-width: 80%;" src="blogs/fxaa/aa_result.png"></center>

<center><p>图2 理想的抗锯齿效果</p></center>

### 像素检测

<center><img style="max-width: 80%;" src="blogs/fxaa/pixel_detective.png"></center>

<center><p>图3 锯齿大多出现在物体边缘（左侧为原图，右侧红色部分为潜在边缘像素）</p></center>

我们通常基于亮度来检测边缘像素，我在shader中计算亮度的公式为 $ L = 0.299 * R + 0.587 * G + 0.114 * B $ 。这里需要解释的是，如果像素亮度可以被提前计算，将会减少FXAA pass的执行时间，一个思路是在Color Grading pass的frame buffer中添加一个颜色附着用来记录亮度。

FXAA采用的边缘检测算法如下：

> 1. 读取上、下、左、右四个临近像素和自身的像素亮度。
> 2. 筛选出其中的亮度的最大值 $maxLuma$ 和最小值 $minLuma$ 。
> 3. 计算对比度 $contrast = maxLuma - minLuma$ 。
> 4. 简单的通过 $contrast$ 是否大于固定阈值 $minThreshold$ 筛选潜在边缘像素会造成画面不够锐利，通过修正新的阈值为 $max(minThreshold, maxLuma * threshold)$ 来避免局部高频信息的丢失。当 $contrast$ 大于阈值时认为该像素是潜在边缘像素，否则直接输出当前像素的颜色值。

### 确定边缘

<center><img style="max-width: 80%;" src="blogs/fxaa/edge_detective.png"></center>

<center><p>图4 对于红点所在像素，绿色箭头为边缘的切线方向，蓝色箭头为边缘的法线方向</p></center>

目前为止我们已经找到了潜在的边缘像素，但是如何确定与哪个邻近像素相混合呢？我们需要确定当前像素所在边缘切线方向和法线方向，切线方向是平行于边缘的方向，法线方向则指向待混合的像素方向。

<center><img style="max-width: 80%;" src="blogs/fxaa/nine_grid.png"></center>

<center><p>图5 当前像素和它临近的八个像素</p></center>

我们可以通过以下规则判断：如果垂直方向的两个临近像素中有一个与当前像素之间的亮度差远大于另外一个与当前像素的亮度差，则认为切线方向是水平方向。为了更加准确的判断边缘的切线，我们在代码中比较了下面两个值的大小来确定切线方向。法线方向则是从当前像素指向了切线方向两侧的两个临近像素与当前像素的亮度差更大的那个像素。

 $$horizontal = |(upleft-left)-(left-downleft)|+2*|(up-center)-(center-down)|+|(upright-right)-(right-downright)|$$

  $$vertical = |(upright-up)-(up-upleft)|+2*|(right-center)-(center-left)|+|(downright-down)-(down-downleft)|$$

### 沿边缘探测终点，确定混合权重

<center><img style="max-width: 80%;" src="blogs/fxaa/points_detective.png"></center>

<center><p>图6 对于红点所在像素的边缘，需要探测两端的终点（蓝点）以确定混合权重</p></center>

首先我们可以将搜索的起始点定为当前像素往法线方向偏移0.5个像素的位置，然后沿切线向两侧探测，直到探测点与起始点的的亮度差大于规定阈值时停止。那么这个规定阈值应该是多少呢？我们可以把阈值设置为0.25倍（这只是一个经验性的系数）待混合的像素与当前像素的亮度差。这时候，混合权重就呼之欲出了：比较起始点到两个终点的距离，较大的距离值作为当前像素的权重，较小的距离值作为待混合像素的权重。

这里需要解释一下的是，如果两个终点离起始点距离太远的话，通常也会忽略处理，因为从视觉角度来说，该边界并没有产生锯齿。除此之外，在进行边缘搜索时，一个一个像素搜索是不现实的，我们可以在搜索时逐渐增大步长来进行优化。

### shader代码

```
#version 310 es

#extension GL_GOOGLE_include_directive : enable

#include "constants.h"
precision highp float;
precision highp int;

layout(set = 0, binding = 0) uniform sampler2D in_color;

layout(location = 0) in vec2 in_uv;

layout(location = 0) out vec4 out_color;


/* pixel index in 3*3 kernel
    +---+---+---+
    | 0 | 1 | 2 |
    +---+---+---+
    | 3 | 4 | 5 |
    +---+---+---+
    | 6 | 7 | 8 |
    +---+---+---+
*/
#define UP_LEFT      0
#define UP           1
#define UP_RIGHT     2
#define LEFT         3
#define CENTER       4
#define RIGHT        5
#define DOWN_LEFT    6
#define DOWN         7
#define DOWN_RIGHT   8
vec2 KERNEL_STEP_MAT[] = vec2[9](
    vec2(-1.0, 1.0), vec2(0.0, 1.0), vec2(1.0, 1.0),
    vec2(-1.0, 0.0), vec2(0.0, 0.0), vec2(1.0, 0.0),
    vec2(-1.0, -1.0), vec2(0.0, -1.0), vec2(1.0, -1.0)
);



/* in order to accelerate exploring along tangent bidirectional, step by an increasing amount of pixels QUALITY(i) 
   the max step count is 12
    +-----------------+---+---+---+---+---+---+---+---+---+---+---+---+
    |step index       | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 |10 |11 |
    +-----------------+---+---+---+---+---+---+---+---+---+---+---+---+
    |step pixels count|1.0|1.0|1.0|1.0|1.0|1.5|2.0|2.0|2.0|2.0|4.0|8.0|
    +-----------------+---+---+---+---+---+---+---+---+---+---+---+---+
*/
#define STEP_COUNT_MAX   12
float QUALITY(int i) {
    if (i < 5) return 1.0;
    if (i == 5) return 1.5;
    if (i < 10) return 2.0;
    if (i == 10) return 4.0;
    if (i == 11) return 8.0;
}


// L = 0.299 * R + 0.587 * G + 0.114 * B
float RGB2LUMA(vec3 color) {
    return dot(vec3(0.299, 0.578, 0.114), color);
}


#define EDGE_THRESHOLD_MIN  0.0312
#define EDGE_THRESHOLD_MAX  0.125
#define SUBPIXEL_QUALITY    0.75
#define GRADIENT_SCALE      0.25

void main()
{
    highp ivec2 screen_size = textureSize(in_color, 0);
    highp vec2 uv_step = vec2(1.0 / float(screen_size.x), 1.0 / float(screen_size.y));

    float luma_mat[9];
    for (int i = 0; i < 9; i++) {
        luma_mat[i] = RGB2LUMA(texture(in_color, in_uv + uv_step * KERNEL_STEP_MAT[i]).xyz);
    }

    // detecting where to apply FXAA, return the pixel color if not
    float luma_min = min(luma_mat[CENTER], min(min(luma_mat[UP], luma_mat[DOWN]), min(luma_mat[LEFT], luma_mat[RIGHT])));
    float luma_max = max(luma_mat[CENTER], max(max(luma_mat[UP], luma_mat[DOWN]), max(luma_mat[LEFT], luma_mat[RIGHT])));
    float luma_range = luma_max - luma_min;
    if(luma_range < max(EDGE_THRESHOLD_MIN, luma_max * EDGE_THRESHOLD_MAX)) {
        out_color = texture(in_color, in_uv);
        return;
    }

    // choosing edge tangent
    // horizontal: |(upleft-left)-(left-downleft)|+2*|(up-center)-(center-down)|+|(upright-right)-(right-downright)|
    // vertical: |(upright-up)-(up-upleft)|+2*|(right-center)-(center-left)|+|(downright-down)-(down-downleft)|
    float luma_horizontal = 
        abs(luma_mat[UP_LEFT] + luma_mat[DOWN_LEFT] - 2.0 * luma_mat[LEFT])
        + 2.0 * abs(luma_mat[UP] + luma_mat[DOWN] - 2.0 * luma_mat[CENTER])
        + abs(luma_mat[UP_RIGHT] + luma_mat[DOWN_RIGHT] - 2.0 * luma_mat[RIGHT]);
    float luma_vertical = 
        abs(luma_mat[UP_LEFT] + luma_mat[UP_RIGHT] - 2.0 * luma_mat[UP])
        + 2.0 * abs(luma_mat[LEFT] + luma_mat[RIGHT] - 2.0 * luma_mat[CENTER])
        + abs(luma_mat[DOWN_LEFT] + luma_mat[DOWN_RIGHT] - 2.0 * luma_mat[DOWN]);
    bool is_horizontal = luma_horizontal > luma_vertical;

    // choosing edge normal 
    float gradient_down_left = (is_horizontal ? luma_mat[DOWN] : luma_mat[LEFT]) - luma_mat[CENTER];
    float gradient_up_right = (is_horizontal ? luma_mat[UP] : luma_mat[RIGHT]) - luma_mat[CENTER];
    bool is_down_left = abs(gradient_down_left) > abs(gradient_up_right);

    // get the tangent uv step vector and the normal uv step vector
    vec2 step_tangent = (is_horizontal ? vec2(1.0, 0.0) : vec2(0.0, 1.0)) * uv_step;
    vec2 step_normal =  (is_down_left ? -1.0 : 1.0) * (is_horizontal ? vec2(0.0, 1.0) : vec2(1.0, 0.0)) * uv_step;

    // get the change rate of gradient in normal per pixel
    float gradient = is_down_left ? gradient_down_left : gradient_up_right;

    // start at middle point of tangent edge
    vec2 uv_start = in_uv + 0.5 * step_normal;
    float luma_average_start = luma_mat[CENTER] + 0.5 * gradient;    

    // explore along tangent bidirectional until reach the edge both
    vec2 uv_pos = uv_start + step_tangent;
    vec2 uv_neg = uv_start - step_tangent;

    float delta_luma_pos = RGB2LUMA(texture(in_color, uv_pos).rgb) - luma_average_start;
    float delta_luma_neg = RGB2LUMA(texture(in_color, uv_neg).rgb) - luma_average_start;

    bool reached_pos = abs(delta_luma_pos) > GRADIENT_SCALE * abs(gradient);
    bool reached_neg = abs(delta_luma_neg) > GRADIENT_SCALE * abs(gradient);
    bool reached_both = reached_pos && reached_neg;

    if (!reached_pos) uv_pos += step_tangent;
    if (!reached_neg) uv_neg -= step_tangent;

    if (!reached_both) {
        for(int i = 2; i < STEP_COUNT_MAX; i++){
            if(!reached_pos) delta_luma_pos = RGB2LUMA(texture(in_color, uv_pos).rgb) - luma_average_start;
            if(!reached_neg) delta_luma_neg = RGB2LUMA(texture(in_color, uv_neg).rgb) - luma_average_start;

            bool reached_pos = abs(delta_luma_pos) > GRADIENT_SCALE * abs(gradient);
            bool reached_neg = abs(delta_luma_neg) > GRADIENT_SCALE * abs(gradient);
            bool reached_both = reached_pos && reached_neg;

            if (!reached_pos) uv_pos += (QUALITY(i) * step_tangent);
            if (!reached_neg) uv_neg -= (QUALITY(i) * step_tangent);

            if (reached_both) break;
        }
    }

    // estimating offset
    float length_pos = max(abs(uv_pos - uv_start).x, abs(uv_pos - uv_start).y);
    float length_neg = max(abs(uv_neg - uv_start).x, abs(uv_neg - uv_start).y);
    bool is_pos_near = length_pos < length_neg;

    float pixel_offset = -1.0 * (is_pos_near ? length_pos : length_neg) / (length_pos + length_neg) + 0.5;

    // no offset if the bidirectional point is too far
    if(((is_pos_near ? delta_luma_pos : delta_luma_neg) < 0.0) == (luma_mat[CENTER] < luma_average_start)) pixel_offset = 0.0;

    // subpixel antialiasing
    float luma_average_center = 0.0;
    float average_weight_mat[] = float[9](
        1.0, 2.0, 1.0,
        2.0, 0.0, 2.0,
        1.0, 2.0, 1.0
    );
    for (int i = 0; i < 9; i++) luma_average_center += average_weight_mat[i] * luma_mat[i];
    luma_average_center /= 12.0;

    float subpixel_luma_range = clamp(abs(luma_average_center - luma_mat[CENTER]) / luma_range, 0.0, 1.0);
    float subpixel_offset = (-2.0 * subpixel_luma_range + 3.0) * subpixel_luma_range * subpixel_luma_range;
    subpixel_offset = subpixel_offset * subpixel_offset * SUBPIXEL_QUALITY;

    // use the max offset between subpixel offset with before
    pixel_offset = max(pixel_offset, subpixel_offset);


    out_color = texture(in_color, in_uv + pixel_offset * step_normal);
}


```