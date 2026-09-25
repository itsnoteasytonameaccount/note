---
title: opengl(5):光照贴图、投光物、多光源
tags:
  - opengl
date: 2021/02/04
---

为学懂webgl，从opengl入手。本文内容主要来自[LearnOpenGL CN](https://learnopengl-cn.github.io/)文档，加上个人理解，以及一些细节补充、经验记录。本人学习过程中的详细代码参见[此处](https://github.com/itsnoteasytonameaccount/OpenGL-learning)

要学习推荐看原文[LearnOpenGL CN](https://learnopengl-cn.github.io/)，有评论区，加载评论区需要翻墙

另可在[此处](https://www.khronos.org/opengl/wiki/Getting_Started)找到其他opengl官方推荐的资料、教程网站
<!-- more -->


# 光照贴图

没什么难搞的地方，直接上代码

```c++
#include "LightingMaps.h"

static glm::mat4 projection;
static int WIDTH = 800, HEIGHT = 600;
static Camera camera(glm::vec3(0.0f, 0.0f, 3.0f));

static const char *vertex_shader_path = "dist/shader/lighting_maps/lighting_maps.vs",
                  *fragment_shader_path = "dist/shader/lighting_maps/lighting_maps.fs",
                  *light_fragment_shader_path = "dist/shader/lighting_maps/lighting_maps_light.fs",
                  *texture_path = "dist/assets/lighting_maps/container2.png",
                  *specular_texture_path = "dist/assets/lighting_maps/container2_specular.png", //"dist/assets/lighting_maps/lighting_maps_specular_color.png",
                  *emission_maps_path = "dist/assets/lighting_maps/matrix.jpg";

static void _setViewport(GLFWwindow *window, int width, int height)
{
    projection = glm::perspective(glm::radians(camera.getZoom()), (float)width / height, 0.1f, 100.0f);
    WIDTH = width;
    HEIGHT = height;
    glViewport(0, 0, width, height);
}

void mouseMove(GLFWwindow *window, double xpos, double ypos)
{
    camera.ProcessMouseMovement(xpos, ypos);
}

void scroll(GLFWwindow *window, double xoffset, double yoffset)
{
    camera.ProcessMouseScroll(yoffset);
}

LightingMaps::LightingMaps(/* args */) : GLWindow(WIDTH, HEIGHT, _setViewport), light_color(glm::vec3(1.0f)), light_pos(glm::vec3(1.2f, 1.0f, 2.0f)), specular(glm::vec3(0.5f, 0.5f, 0.5f)), box(1), shininess(32.0f)
{

    this->mouseCallback = mouseMove;
    this->scrollCallback = scroll;

    initWindow(WIDTH, HEIGHT, "材质场景");
    glEnable(GL_DEPTH_TEST);
    glClearColor(0.0f, 0.0f, 0.0f, 1.0f);
    glClearDepth(1.0f);

    object_shader.readFile(vertex_shader_path, fragment_shader_path);
    light_shader.readFile(vertex_shader_path, light_fragment_shader_path);

    box.createVAO();

    texture0num = 0;
    texture1num = 1;
    texture2num = 2;
    Texture::LoadTexture(texture_path, &texture0, 1, texture0num);
    Texture::LoadTexture(specular_texture_path, &texture1, 1, texture1num);
    Texture::LoadTexture(emission_maps_path, &texture2, 1, texture2num);
}

LightingMaps::~LightingMaps()
{
}

void LightingMaps::draw()
{
    glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);

    glm::mat4 model(1.0f);
    glm::mat4 view = camera.getViewMatrix();

    // light_color.x = sin(getTime() * 2.0f);
    // light_color.y = sin(getTime() * 0.7f);
    // light_color.z = sin(getTime() * 1.3f);

    object_shader.useProgram();
    object_shader.setUniform3fv("light.position", glm::vec3(view * glm::vec4(light_pos, 1.0f)));
    object_shader.setUniform3fv("light.ambient", light_color * glm::vec3(0.2f));
    object_shader.setUniform3fv("light.diffuse", light_color * glm::vec3(0.5f));
    object_shader.setUniform3fv("light.specular", glm::vec3(1.0f));
    object_shader.setUniform1i("material.diffuse", texture0num);
    object_shader.setUniform1i("material.specular", texture1num);
    object_shader.setUniform1i("emissionMaps", texture2num);
    object_shader.setUniform1f("material.shininess", shininess);
    object_shader.setUniformMatrix4("projection", projection);
    object_shader.setUniformMatrix4("model", model);
    object_shader.setUniformMatrix4("view", view);
    object_shader.setUniformMatrix3("normalMatrix", glm::mat3(glm::transpose(glm::inverse(view * model))));
    box.draw();

    light_shader.useProgram();
    model = glm::translate(model, light_pos);
    model = glm::scale(model, glm::vec3(0.5f, 0.5f, 0.5f));
    light_shader.setUniformMatrix4("projection", projection);
    light_shader.setUniformMatrix4("model", model);
    light_shader.setUniformMatrix4("view", view);
    light_shader.setUniform3fv("lightColor", light_color);
    box.draw();
}

void LightingMaps::getKeyInput()
{
    int direction = 0x00;
    if (getKeyPress(GLFW_KEY_ESCAPE))
    {
        setShouldClose(true);
    }
    if (getKeyPress(GLFW_KEY_UP, GLFW_PRESS))
    {
        direction |= _CAMERA_UP;
    }
    if (getKeyPress(GLFW_KEY_DOWN))
    {
        direction |= _CAMERA_DOWN;
    }
    if (getKeyPress(GLFW_KEY_RIGHT))
    {
        direction |= _CAMERA_RIGHT;
    }
    if (getKeyPress(GLFW_KEY_LEFT))
    {
        direction |= _CAMERA_LEFT;
    }
    if (getKeyPress(GLFW_KEY_W))
    {
        direction |= _CAMERA_FORWARD;
    }
    if (getKeyPress(GLFW_KEY_S))
    {
        direction |= _CAMERA_BACKWARD;
    }
    if (direction)
    {
        camera.ProcessKeyboardInput(direction, delta);
    }
}
```

顶点着色器

```glsl
#version 330 core

layout (location = 0) in vec3 aPos;
layout (location = 1) in vec3 aNormal;
layout (location = 2) in vec2 aTexCoords;

uniform mat4 projection;
uniform mat4 view;
uniform mat4 model;
uniform mat3 normalMatrix;

out vec2 TexCoords;
out vec3 Normal;
out vec3 FragPos;

void main()
{
    gl_Position = projection * view * model * vec4(aPos, 1.0f);
    FragPos = vec3(view * model * vec4(aPos, 1.0f));
    Normal = vec3(normalMatrix * aNormal);
    TexCoords = aTexCoords;
}
```

片段着色器

```glsl
#version 330 core

struct Material
{
    sampler2D diffuse;
    sampler2D specular;
    float shininess;
};

struct Light
{
    vec3 position;
    vec3 diffuse;
    vec3 ambient;
    vec3 specular;
};

uniform Material material;
uniform Light light;
uniform sampler2D emissionMaps;

in vec3 FragPos;
in vec3 Normal;
in vec2 TexCoords;

out vec4 FragColor;

void main()
{
    vec3 ambient = light.ambient * vec3(texture(material.diffuse, TexCoords));

    vec3 lightDir = normalize(light.position - FragPos);
    float diff = max(dot(lightDir, Normal), 0.0f);
    vec3 diffuse = light.diffuse * diff * vec3(texture(material.diffuse, TexCoords));

    vec3 viewDir = normalize(-FragPos);
    vec3 reflectDir = reflect(-lightDir, Normal);
    float spec = pow(max(dot(reflectDir, viewDir), 0.0f), material.shininess);
    vec3 specular = light.specular * spec *  vec3(texture(material.specular, TexCoords));
    // vec3 specular = light.specular * spec *  (1 - vec3(texture(material.specular, TexCoords)));

    vec3 emission = vec3(1.0f) *  vec3(texture(emissionMaps, TexCoords));

    FragColor = vec4(specular + diffuse + ambient + emission, 1.0f);
}

```

光源立方体的顶点着色器

```glsl
#version 330 core

out vec4 FragColor;

void main()
{
    FragColor = vec4(1.0f);
}
```

# 投光物

> 根据本节中的代码可以看到目前所有光照均未考虑不同物体之间遮挡光线的情况。

## 平行光

平行光比较简单，光线方向向量不需要计算，而是直接从程序中传入片段着色器，在所有位置，光的方向向量都是一致的。

![图片无法显示](../image/opengl/light_casters_directional.png)

根据 opengl 的提示，我们尝试将所有向量改为四分量的，并根据 w 分量是否为 1.0f 判断向量代表光线的位置还是光线的方向以区分光线是点光源或者平行光，修改之后

顶点着色器

```glsl
#version 330 core
layout (location = 0) in vec3 aPos;
layout (location = 1) in vec3 aNormal;
layout (location = 2) in vec2 aTexCoords;

uniform mat4 projection;
uniform mat4 view;
uniform mat4 model;
uniform mat3 normalMatrix;

out vec4 Normal;
out vec4 FragPos;
out vec2 TexCoords;

void main()
{
    FragPos = view * model * vec4(aPos, 1.0f);
    gl_Position = projection * FragPos;

    Normal = vec4(normalize(normalMatrix * aNormal), 0.0f);
    TexCoords = aTexCoords;
}
```

片段着色器

```glsl
#version 330 core
in vec2 TexCoords;
in vec4 Normal;
in vec4 FragPos;

struct Light
{
    vec4 vector;
    // ...
};

struct Material
{
    sampler2D diffuse;
    sampler2D specular;
    float shininess;
};

uniform Material material;
uniform Light light;

out vec4 FragColor;

void main()
{
    // ...
    if (light.vector.w == 0.0f)
    {
        // 平行光
        lightDir = normalize(-light.vector);
    }
    else if (light.vector.w == 1.0f)
    {
        // 点光源
        lightDir = normalize(light.vector - FragPos);
    }
    // ...
}
```

c++代码中

```c++
object_shader.setUniform4fv("light.vector", view * light_pos);
```

## 点光源

点光源光线强度应当随着距离增加而衰减

![图片无法显示](../image/opengl/light_casters_point.png)

因此需要计算点光源的与物体表面点的距离，用向量长度即可，glsl 提供了向量长度计算公式。

另外还需要一个衰减公式来计算衰减强度，理想的衰减公式应当是在靠近点光源处光线强度随着距离增大变化较快，在远离点光源处光线强度随着距离增大变化较慢，因此公式不应当是线性的

现成的公式

$$

{\large F}_{att} = \frac{1.0}{K_c + K_l * d + K_q * d^2}

$$

其中

- $d$: 光源到物体表面的距离
- ${\large K}_c$: 常数项，用于避免分母小于 1，所以一般取 1
- ${\large K}_l$: 一次项系数，一定距离内当${\large K}_l \gg 2{\large K}_q*d$，一次项对光线强度改变起主导作用
- ${\large K}_q$: 二次项系数，一定距离后${\large K}_l \ll 2{\large K}_q*d$，二次项对光线强度改变起主导作用

${\large K}_c,{\large K}_l,{\large K}_q$三者取值及效果表格如下

| 光线覆盖距离 | ${\large K}_c$ | ${\large K}_l$ | ${\large K}_q$ |
| :----------- | :------------- | :------------- | :------------- |
| 7            | 1.0            | 0.7            | 1.8            |
| 13           | 1.0            | 0.35           | 0.44           |
| 20           | 1.0            | 0.22           | 0.20           |
| 32           | 1.0            | 0.14           | 0.07           |
| 50           | 1.0            | 0.09           | 0.032          |
| 65           | 1.0            | 0.07           | 0.017          |
| 100          | 1.0            | 0.045          | 0.0075         |
| 160          | 1.0            | 0.027          | 0.0028         |
| 200          | 1.0            | 0.022          | 0.0019         |
| 325          | 1.0            | 0.014          | 0.0007         |
| 600          | 1.0            | 0.007          | 0.0002         |
| 3250         | 1.0            | 0.0014         | 0.000007       |

在片段着色器中视线衰减如下

```glsl
// ...
struct Light {
    vec3 vector;  

    vec3 ambient;
    vec3 diffuse;
    vec3 specular;

    float constant;
    float linear;
    float quadratic;
};

void main()
{
    // ...
    
    float distance = 0.0f, attenuation = 1.0f;
    if (light.vector.w == 0.0f)
    {
        lightDir = normalize(-light.vector);
    } 
    else if (light.vector.w == 1.0f) 
    {
        lightDir = light.vector - FragPos;
        // 内置length函数可以求向量的模
        distance = length(lightDir);
        lightDir = normalize(lightDir);
        attenuation = 1.0f / (light.constant + light.linear * distance + light.quadratic * distance * distance); 
    }
    // ...

    result = ambient + diffuse + specular;
    FragColor = vec4(result.xyz * attenuation, 1.0f);
}
```

> 问题：点光源这一部分引出了一个问题，既然点光源的光在不同距离点处有不同程度的衰减，那么物体漫反射及镜面反射到摄像机前的所有光是否也应该计算一下衰减值（摄像机与某点的距离和点光源到某点的距离是不同的）？计算了会不会显得更真实点？

## 聚光

聚光为有一定范围的点光源，超过范围只有环境光起作用

![图片无法显示](../image/opengl/light_casters_spotlight_angles.png)

计算聚光需要的增加参数有：聚光锥体的中轴线方向SpotDir；聚光边缘光线方向向量与中轴的夹角，切光角$\phi$，用弧度表示；

只要光线与中轴的夹角$\theta$超过切光角$\phi$便不计算镜面光照和漫反射。

弧度通过余弦值比较，余弦值通过单位向量点乘来求，即判断方式为

```glsl
if(dot(-lightDir, SpotDir) < light.cutOff)
{
    result = ambient;
}
else
{
    // ...

    result = ambient + specular + diffuse;
}
```

但是仅进行如上操作会导致聚光边缘分界明显，不够真实，需要在边缘处进行渐变让光线渐渐变暗

因此添加一个外切光角$\gamma$，在内切光角内光线强度不随$\theta$变化。

内切光角到外且光角之间光线逐渐变暗，渐变公式

$$
{\large I} = \frac{\cos\theta - \cos\gamma}{\cos\phi - \cos\gamma}
$$
可以看到

$$
\begin{cases}
{\large I} < 0 &,\theta \gt \gamma \\
{\large I} \in [0, 1]&,\theta \in [\phi,\gamma]\\
{\large I} > 1&,\theta \lt \phi
\end{cases}
$$

当$\theta > \gamma$时应当只有环境光

修改后聚光在片段着色器中的计算

```glsl
// ...

void main()
{
    // ...

    float theta = dot(-lightDir, normalize(light.direction));
    // clamp函数的作用是将点乘结果限制在一定范围内，如下代码对小于0的取0，大于1的取1
    float intensity =  clamp((theta - light.outerCutOff)/(light.cutOff - light.outerCutOff), 0.0f, 1.0f);

    // ...
    
    result = vec4((specular + diffuse).xyz * intensity, 1.0f) + ambient;

    FragColor = vec4(result.xyz * attenuation, 1.0f);
}
```

## 总结代码

片段着色器

```glsl
#version 330 core
in vec2 TexCoords;
in vec4 Normal;
in vec4 FragPos;

struct Light
{
    vec4 vector;
    vec4 direction;
    float cutOff;
    float outerCutOff;

    vec4 position;
    vec4 ambient;
    vec4 specular;
    vec4 diffuse;
    
    float constant;
    float linear;
    float quadratic;
};

struct Material
{
    sampler2D diffuse;
    sampler2D specular;
    float shininess;
};

uniform Material material;
uniform Light light;

out vec4 FragColor;

void main()
{
    vec4 ambient = light.ambient * texture(material.diffuse, TexCoords);
    
    vec4 lightDir, result;
    float distance = 0.0f, attenuation = 1.0f;

    // 平行光
    if (light.vector.w == 0.0f)
    {
        lightDir = normalize(-light.vector);
    } 
    // 点光
    else if (light.vector.w == 1.0f) 
    {
        lightDir = light.vector - FragPos;
        distance = length(lightDir);
        lightDir = normalize(lightDir);
        // 点光衰减
        attenuation = 1.0f / (light.constant + light.linear * distance + light.quadratic * distance * distance); 
    }

    // 聚光
    float theta = dot(-lightDir, normalize(light.direction));
    float intensity =  clamp((theta - light.outerCutOff)/(light.cutOff - light.outerCutOff), 0.0f, 1.0f);

    float diff = max(dot(lightDir, Normal), 0.0f);
    vec4 diffuse = light.diffuse * diff * texture(material.diffuse, TexCoords);

    vec4 viewDir = normalize(vec4(0.0f, 0.0f, 0.0f, 1.0f) - FragPos); 
    vec4 reflectDir = reflect(-lightDir, Normal);
    float spec = pow(max(dot(reflectDir, viewDir), 0.0f), material.shininess);
    vec4 specular = light.specular * spec * texture(material.specular, TexCoords);

    result = vec4((specular + diffuse).xyz * intensity, 1.0f) + ambient;

    FragColor = vec4(result.xyz * attenuation, 1.0f);
}
```

# 多光源

使用多个不同光源时可将各自的计算方式封装为函数

建议learn openglcn提供了分别封装的代码，本文出于练手的目的将所有光源抽象为一个结构体

片段着色器如下

```glsl
// ...

struct Light
{
    vec4 vector;
    vec4 direction;
    float cutOff;
    float outerCutOff;

    vec4 ambient;
    vec4 specular;
    vec4 diffuse;
    
    float constant;
    float linear;
    float quadratic;
};

// ...

vec3 CalcLight(Light light, vec4 normal, vec4 viewDir, vec4 diffuseTexture, vec4 specularTexture)
{
    vec4 ambient = light.ambient * diffuseTexture;

    vec4 lightDir;
    float distance, attenuation = 1.0f;
    // 分量为1代表位置向量
    if(light.vector.w == 1.0f)
    {
        // 点光源或聚光计算衰减
        lightDir = light.vector - FragPos;
        distance = length(lightDir);
        attenuation = 1.0f / (light.constant + light.linear * distance + light.quadratic * distance *distance);
        lightDir = normalize(lightDir);
    }
    else
    {
        // 平行光
        lightDir = normalize(-light.vector);
    }

    // 聚光
    float intensity = 0.0f, theta = dot(light.direction, -lightDir);
    // 内切光角范围内
    if(theta >= light.cutOff)
    {
        intensity = 1.0f;
    }
    // 外切光角范围内
    else if(theta >= light.outerCutOff)
    {
        intensity = (theta - light.outerCutOff)/(light.cutOff - light.outerCutOff);
    }

    float diff = max(dot(lightDir, normal), 0.0f);
    vec4 diffuse = light.diffuse * diff * diffuseTexture;

    vec4 reflectDir = reflect(-lightDir, normal);
    float spec = pow(max(dot(reflectDir, viewDir), 0.0f), material.shininess);
    vec4 specular = light.specular * spec * specularTexture;

    vec3 result = (diffuse + specular).xyz * intensity + ambient.xyz;
    return result.xyz * attenuation;
}

// ...

```

c++ 代码中

```c++
// 复制字符串
/**
 * @param {char*} dst 目标数组
 * @param {char*} src 源数据
 * @param {int} start 复制到目标数组的start位置之后
 * @return {int} 返回复制的字符串长度
 */
int str_copy(char *dst, char const *src, int start)
{
    char *p1 = dst + start;
    char const *p2 = src;
    while (*p2 != 0)
    {
        *p1++ = *p2++;
    }
    *p1 = 0;
    return p1 - dst;
}

// 设置光线结构体
void LightCasters::setLight(char const *name, char index, glm::vec4 vector, glm::vec4 color, int type)
{
    // 设置参数类型
    int len = str_copy(param, name, 0);
    // 不同类型的光系数不同
    float ambient = 0.2, diffuse = 0.5, specular = 1.0f;
    if (index)
    {
        param[len++] = '[';
        param[len++] = index;
        param[len++] = ']';
    }
    param[len++] = '.';

    str_copy(param, "specular", len);
    object_shader.setUniform4f(param, color.x, color.y, color.z, 1.0f);

    str_copy(param, "linear", len);
    object_shader.setUniform1f(param, 0.09f);
    str_copy(param, "quadratic", len);
    object_shader.setUniform1f(param, 0.032f);
    str_copy(param, "constant", len);
    object_shader.setUniform1f(param, 1.0f);

    // object_shader.setUniform4f(param, )
    switch (type)
    {
    case DIR_LIGHT:
        str_copy(param, "cutOff", len);
        // 内切光角设为-1时光线全局有效
        object_shader.setUniform1f(param, -1.0f);
        ambient = 0.05f;
        diffuse = 0.4f;
        specular = 0.5f;

        str_copy(param, "vector", len);
        object_shader.setUniform4f(param, vector.x, vector.y, vector.z, vector.w);
        break;
    case SPOT_LIGHT:
        str_copy(param, "vector", len);
        object_shader.setUniform4f(param, vector.x, vector.y, vector.z, 1.0f);
        str_copy(param, "direction", len);
        object_shader.setUniform4f(param, 0.0f, 0.0f, -1.0f, 0.0f);

        ambient = 0.0f;
        diffuse = 1.0f;
        specular = 1.0f;

        str_copy(param, "cutOff", len);
        object_shader.setUniform1f(param, glm::cos(glm::radians(12.5f)));
        str_copy(param, "outerCutOff", len);
        object_shader.setUniform1f(param, glm::cos(glm::radians(17.5f)));
        break;
    case POINT_LIGHT:
        str_copy(param, "cutOff", len);
        object_shader.setUniform1f(param, -1.0f);

        ambient = 0.05f;
        diffuse = 0.8f;
        specular = 1.0f;

        str_copy(param, "vector", len);
        object_shader.setUniform4f(param, vector.x, vector.y, vector.z, 1.0f);
        break;
    default:
        break;
    }

    str_copy(param, "ambient", len);
    object_shader.setUniform4f(param, ambient * color.x, ambient * color.y, ambient * color.z, 1.0f);
    str_copy(param, "diffuse", len);
    object_shader.setUniform4f(param, diffuse * color.x, diffuse * color.y, diffuse * color.z, 1.0f);
}

void LightCasters::draw()
{
    // ...
    object_shader.useProgram();
    // 聚光
    setLight("spotLight", 0, glm::vec4(0.0f, 0.0f, 0.0f, 1.0f), light_color, SPOT_LIGHT);
    // 平行光
    setLight("dirLight", 0, view * light_direction, light_color, DIR_LIGHT);
    
    // 点光
    for (int i = 0; i < 4; i++)
    {
        setLight("lights", (char)(i + 48), view * glm::vec4(pointLightPositions[i], 1.0f), light_color, POINT_LIGHT);
    }
    // ...
}
```

# 参考

1. [learnOpenGL CN](https://learnopengl-cn.github.io/02%20Lighting/06%20Multiple%20lights/)