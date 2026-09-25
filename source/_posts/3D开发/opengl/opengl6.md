---
title: opengl(6):模型加载
tags:
- opengl
date: 2021/02/08
---

为学懂webgl，从opengl入手。本文内容主要来自[LearnOpenGL CN](https://learnopengl-cn.github.io/)文档，加上个人理解，以及一些细节补充、经验记录。本人学习过程中的详细代码参见[此处](https://github.com/itsnoteasytonameaccount/OpenGL-learning)

要学习推荐看原文[LearnOpenGL CN](https://learnopengl-cn.github.io/)，有评论区，加载评论区需要翻墙

另可在[此处](https://www.khronos.org/opengl/wiki/Getting_Started)找到其他opengl官方推荐的资料、教程网站
<!-- more -->

# wavefront的obj文件格式

了解一下wavefront的obj文件格式。推荐的[3d模型下载网站](https://free3d.com/)

obj文件为文本，内容格式如下

```bash
# 带'#'号的为注释
# 顶点的列表，每个顶点占一行，用v开头，v后跟随3个数字分别表示右手坐标系的(x,y,z)坐标，第四个w分量可选，默认为1
v 0.123 0.234 0.345 1.0
v ...
...
# 纹理坐标列表，vt开头，使用(u,v,w)坐标，取值0-1，v，w可选，默认为0
vt 0.500 1 [0]
vt ...
...
# 顶点法向量列表，xyz坐标，可能不是单位向量
vn 0.707 0.000 0.707
vn ...
...
# 参数空间顶点列表（Parameter space vertices,了解有限，不知道这是什么可能翻译有错），uvw坐标，定义参数空间中曲线或面上的点
vp 0.310000 3.210000 2.100000
vp ...
...
# 多边形表面元素，以f开头，后面跟随的每组数字为顶点的各项要素坐标索引，格式为v[/[vt]/vn]，每个表面后跟随3到n组这样的数字
# 数值为正是顺数，为负是倒数
f 1 2 3 #即该面由第1、2、3个顶点组成
f 3/1 4/2 -1/3 #即该面由第3、4、以及最后一个顶点组成，各个顶点的纹理坐标分别取第1、2、3个纹理坐标
f 6/4/1 3/5/3 7/6/5 #即该面由第6、3、7个顶点组成，各个顶点的纹理坐标分别取第4、5、6个纹理坐标，法向量分别取第1、3、5个法向量
f 7//1 8//2 9//3 #即该面由第7、8、9个顶点组成，法向量分别取第1、2、3个法向量
f ...
```

## 材质引用

材质被定义在外部的.mtl文件中，如下方式引用mtl文件及使用材质

```bash
# 引用mtl文件
mtllib [mymaterial.mtl]
...

# 使用材质，material name在上面引用的mtl文件中定义
usemtl [material name]
```
## 其他标签

如下标签可指定对象、组或启用[平滑着色(smooth shading)](https://en.wikipedia.org/wiki/Shading)等

```bash
# 命名对象和组的方式
o [object name]
  ...
  g [group name]
  ...

# 多边形的平滑着色(smooth shading)通过平滑组启用
s 1
  ...
  # Smooth shading can be disabled as well.
  s off
  ...
```

## 相对和绝对索引

上文以f开头定义的多边形表面中正数定义的索引是绝对索引，负数定义的是相对索引，可能在不同软件中存在不兼容相对索引或只使用相对索引的情况。

## mtl(Material template library)文件

```bash
# 环境
Ka 1.000 1.000 1.000
# 漫反射
Kd 1.000 1.000 1.000
# 镜面
Ks 1.000 1.000 1.000
# 镜面高光亮度，范围0-1000
Ns 10.000
# 透明，不像真实的透明度，这个选项的结果不被物体的厚度影响
d 0.9
# 透明度，同上，不过Tr = 1 - d
Tr 0.1
# 设置了透明的材质还可以设置一个传播过滤色(Transmission Filter Color)
# RGB格式
Tf 1.0 0.5 0.5
# CIEXYZ格式，yz可以忽略，忽略时默认等于x
Tf xyz 1.0 0.5 0.5 
# 光学容度(optical dencity)，或者说折射率，玻璃一般是1.5，取1.0时不折射，小于1.0时产生怪异的结果不建议使用
Ni 1.45000
```

多种照明模型可选，实现透明度不需要设置透明照明模型，并且现在的用法中常常不指定照明模型，下面因为有些部分暂时没学，保留原文

0. Color on 
1. Color on and Ambient on
2. Highlight on
3. Reflection on and Ray trace on
4. Transparency: Glass on, Reflection: Ray trace on
5. Reflection: Fresnel on and Ray trace on
6. Transparency: Refraction on, Reflection: Fresnel off and Ray trace on
7. Transparency: Refraction on, Reflection: Fresnel on and Ray trace on
8. Reflection on and Ray trace off
9. Transparency: Glass on, Reflection: Ray trace off
10. Casts shadows onto invisible surfaces

```bash
# 指定光照模型
illum 2
```

纹理材质

```bash
newmtl Textured
   Ka 1.000 1.000 1.000
   Kd 1.000 1.000 1.000
   Ks 0.000 0.000 0.000
   d 1.0
   illum 2
   # the ambient texture map
   map_Ka lemur.tga
   
   # the diffuse texture map (most of the time, it will be the same as the
   # ambient texture map)
   map_Kd lemur.tga
   
   # specular color texture map
   map_Ks lemur.tga
   
   # specular highlight component
   map_Ns lemur_spec.tga
   
   # the alpha texture map
   map_d lemur_alpha.tga
   
   # some implementations use 'map_bump' instead of 'bump' below
   map_bump lemur_bump.tga
   
   # bump map (which by default uses luminance channel of the image)
   bump lemur_bump.tga
   
   # displacement map
   disp lemur_disp.tga
   
   # stencil decal texture (defaults to 'matte' channel of the image)
   decal lemur_stencil.tga
```

# assimp

> 本人用的编译工具是mingw64

assimp是一个用来加载模型的库，接下来下载编译assimp，方法和glfw一样，用cmake-gui配置好路径后第一次configure选择编译器并生成可选参数，第二次configure设置参数，最后generate输出文件，然后编译

编译时有个选项`BUILD_SHARED_LIBS`默认是勾选的，编译后生成的是动态库，需要放到合适的路径。

__碰到的错误__

编译时出了如下错误，似乎是链接什么可执行文件时出错了，但仍在`输出文件夹/tools/assimp_cmd`和`输出文件夹/code`下生成了libassimp.dll，但似乎没法使用。

![图片无法显示](../image/opengl/compile_error.png)

使用的c编译器是mingw64的gcc.exe，c++编译器是mingw64的g++.exe。

> 解决方案

不要勾选编译动态库选项，生成的libassimp.a可以使用，依赖contrib目录生成的IrrXML和zlib库。不过因为是静态库，程序编译时要链接这个库的话会比较耗时，而且用`ld`命令把静态文件链接成动态文件后，得到的也仍然是不可用的动态文件。

这个问题似乎是[Bug](https://github.com/assimp/assimp/issues/3261)，仍没有解决

## assimp加载的模型

assimp加载得到其定义的场景数据结构，
具体结构如下图
+ 场景对象包括一个树状结构存储节点列表，一个网格列表，一个材质列表
+ 每个节点存储了一个网格的索引列表，以及子节点列表
+ 网格则包括顶点、顶点法向量、材质索引、纹理坐标

![图片无法显示](../image/opengl/assimp_structure.png)

## model类

可以根据结点的结构自行编写加载模型的model类，将assimp数据结构转化为自己操作方便的结构，或者直接用assimp的数据结构绘图。
> 注意：目前可能用不上模型的节点父子关系，可以直接遍历scene的mesh列表来绘制，但最好在自己设置的结构中对节点结构有所体现，方便之后对模型特定部位进行坐标变换。

[learnOpenGL CN](https://learnopengl-cn.github.io/03%20Model%20Loading/01%20Assimp/)提供了参考代码。

另外想找自己需要用到的函数等，可以在assimp的头文件中找。

部分实现如下（来自learnOpenGL CN，根据自己需要略作修改）

__Mesh.cpp__

```c++
// 网格类
Mesh::Mesh(vector<mesh::Vertex> vertices, vector<unsigned int> indices, vector<mesh::Texture> textures, mesh::Material material, bool has_texture)
{
    // 顶点
    this->vertices = vertices;
    // 索引
    this->indices = indices;
    // 材质纹理列表
    this->textures = textures;
    // 材质颜色
    this->material = material;
    // 材质是否是纹理
    this->has_texture = has_texture;

    setupMesh();
}

// ...

void Mesh::draw(Shader &shader)
{
    unsigned int specular_index = 1, diffuse_index = 1;
    std::string name, number;
    if (has_texture)
    {
        shader.setUniform1i("material.has_texture", 1);
        for (unsigned int i = 0; i < textures.size(); i++)
        {
            glActiveTexture(GL_TEXTURE0 + i);
            name = textures[i].type;
            if (name == "texture_diffuse")
            {
                number = std::to_string(diffuse_index++);
            }
            else if (name == "texture_specular")
            {
                number = std::to_string(specular_index++);
            }
            shader.setUniform1i(("material." + name + number).c_str(), i);
            glBindTexture(GL_TEXTURE_2D, textures[i].id);
        }
    }
    else
    {
        shader.setUniform1i("material.has_texture", 0);
        shader.setUniform3f("material.specular", material.specular.r, material.specular.g, material.specular.b);
        shader.setUniform3f("material.ambient", material.ambient.r, material.ambient.g, material.ambient.b);
        shader.setUniform3f("material.diffuse", material.diffuse.r, material.diffuse.g, material.diffuse.b);
    }
    glActiveTexture(GL_TEXTURE0);
    glBindVertexArray(VAO);
    glDrawElements(GL_TRIANGLES, indices.size(), GL_UNSIGNED_INT, 0);
    glBindVertexArray(0);
}
```

__Model.cpp__

```c++
// ...

void Model::loadModel(std::string path)
{
    std::cout << "start importer" << std::endl;
    Assimp::Importer importer;
    const aiScene *scene = importer.ReadFile(path, aiProcess_Triangulate | aiProcess_FlipUVs);
    if (!scene || scene->mFlags & AI_SCENE_FLAGS_INCOMPLETE || !scene->mRootNode)
    {
        std::cout << "ERROR::ASSIMP::" << importer.GetErrorString() << std::endl;
        return;
    }
    directory = path.substr(0, path.find_last_of('/') + 1);
    std::cout << "start node processor" << std::endl;
    processNode(scene->mRootNode, scene);
}

void Model::processNode(aiNode *node, const aiScene *scene)
{
    for (unsigned int i = 0; i < node->mNumMeshes; i++)
    {
        aiMesh *mesh = scene->mMeshes[node->mMeshes[i]];
        meshes.push_back(processMesh(mesh, scene));
    }

    for (unsigned int i = 0; i < node->mNumChildren; i++)
    {
        processNode(node->mChildren[i], scene);
    }
}

Mesh Model::processMesh(aiMesh *mesh, const aiScene *scene)
{
    vector<mesh::Vertex> vertices;
    vector<unsigned int> indices;
    // 网格纹理列表
    vector<mesh::Texture> textures;
    // 材质结构体，包含环境、镜面、漫反射颜色
    mesh::Material mesh_material;
    // 标志位
    bool has_texture = false;

    for (unsigned int i = 0; i < mesh->mNumVertices; i++)
    {
        mesh::Vertex vertex;
        vertex.Position.x = mesh->mVertices[i].x;
        vertex.Position.y = mesh->mVertices[i].y;
        vertex.Position.z = mesh->mVertices[i].z;
        if (mesh->HasNormals())
        {
            vertex.Normal.x = mesh->mNormals[i].x;
            vertex.Normal.y = mesh->mNormals[i].y;
            vertex.Normal.z = mesh->mNormals[i].z;
        }
        if (mesh->mTextureCoords[0])
        {
            vertex.TexCoord.x = mesh->mTextureCoords[0][i].x;
            vertex.TexCoord.y = mesh->mTextureCoords[0][i].y;
        }
        else
        {
            vertex.TexCoord = glm::vec2(0.0f);
        }
        vertices.push_back(vertex);
    }

    for (unsigned int i = 0; i < mesh->mNumFaces; i++)
    {
        aiFace face = mesh->mFaces[i];
        for (unsigned int j = 0; j < face.mNumIndices; j++)
        {
            indices.push_back(face.mIndices[j]);
        }
    }

    if (mesh->mMaterialIndex >= 0)
    {
        aiMaterial *material = scene->mMaterials[mesh->mMaterialIndex];
        aiString material_name = material->GetName();
        unsigned int len = material->GetTextureCount(aiTextureType_DIFFUSE);
        // 纹理作为材质时
        if (len > 0)
        {
            has_texture = true;
            loadMaterialTextures(material, aiTextureType_DIFFUSE, "texture_diffuse", textures, len);
            loadMaterialTextures(material, aiTextureType_SPECULAR, "texture_specular", textures, len);
        }
        // 颜色作为材质时
        else
        {
            aiColor3D diffuse, specular, ambient;
            material->Get(AI_MATKEY_COLOR_DIFFUSE, diffuse);
            mesh_material.diffuse = diffuse;
            material->Get(AI_MATKEY_COLOR_SPECULAR, specular);
            mesh_material.specular = specular;
            material->Get(AI_MATKEY_COLOR_AMBIENT, ambient);
            mesh_material.ambient = ambient;
        }
    }

    // 创建匿名网格对象
    // 编译时只要启用了优化，返回对象似乎不会对性能有影响
    return Mesh(vertices, indices, textures, mesh_material, has_texture);
}
// ...
```

成果如下，从[free3d](https://free3d.com/)下载的obj格式的钢铁侠模型，虽然似乎有些地方材质颜色加载失败

![图片无法显示](../image/opengl/model_loading.gif)

# 参考

1. [wikipedia:Wavefront .obj file](https://en.wikipedia.org/wiki/Wavefront_.obj_file)
2. [learnOpenGL CN](https://learnopengl-cn.github.io/03%20Model%20Loading/01%20Assimp/)