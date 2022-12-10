# OpenGL

## CLion 中配置 opengl 环境

在 Windows 下开发，使用 [glfw](https://github.com/glfw/glfw), [glew](https://glew.sourceforge.net/) 库，工具链是 VS 。

下载这些库后需要把 Include, DLL 和 Lib 放到 project dir 的对应目录下。

```
find_package(OpenGL REQUIRED)

add_library(glfw SHARED IMPORTED)
set_property(TARGET glfw PROPERTY IMPORTED_LOCATION "${PROJECT_SOURCE_DIR}/glfw3.dll")
set_property(TARGET glfw PROPERTY IMPORTED_IMPLIB "${PROJECT_SOURCE_DIR}/Lib/glfw3.lib")
add_library(glew SHARED IMPORTED)
set_property(TARGET glew PROPERTY IMPORTED_LOCATION "${PROJECT_SOURCE_DIR}/glew32.dll")
set_property(TARGET glew PROPERTY IMPORTED_IMPLIB "${PROJECT_SOURCE_DIR}/Lib/glew32.lib")

include_directories(Include)
add_executable(learn_cg main.cpp)
target_link_libraries(learn_cg
        ${OPENGL_LIBRARY} # filled by "find_package(OpenGL REQUIRED)"
        glfw glew
        )
```

另外运行的工作目录也要指定为 project dir

> 可以考虑研究一下 vcpkg

## shader 和 program

创建 shader 的步骤：读取 shader 源码，编译。

```cpp
    unsigned int vertexShader = glCreateShader(GL_VERTEX_SHADER);
    // id, source 数量, source 数组指针，source 长度
    glShaderSource(vertexShader, 1, &vertexShaderSource, NULL);
    glCompileShader(vertexShader);
```

之后要和 Program 链接，注意之前传的源码 buffer 不能被销毁，否则会出错。

```cpp
    shaderProgram = glCreateProgram();
    glAttachShader(shaderProgram, vertexShader);
    glAttachShader(shaderProgram, fragmentShader);
    glLinkProgram(shaderProgram);
```

Program 创建完成后，可以把 shader 删除了。

```cpp
    glDeleteShader(vertexShader);
    glDeleteShader(fragmentShader);
```

之后我们可以用 program 的 id 去访问 shaders 中的 uniform 变量，并使用 shader 绘制顶点 (`glUseProgram`)。

## VAO, VBO, EBO

```cpp
GLuint VAO, VBO, EBO;
static void CreateVertexBuffer() {
    glGenVertexArrays(1, &VAO);
    glGenBuffers(1, &VBO);
    glGenBuffers(1, &EBO);
    glBindVertexArray(VAO);
    glBindBuffer(GL_ARRAY_BUFFER, VBO);
    glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW);
    glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, EBO);
    glBufferData(GL_ELEMENT_ARRAY_BUFFER, sizeof(indices), indices, GL_STATIC_DRAW);
    glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 6 * sizeof(GLfloat), nullptr);
    glEnableVertexAttribArray(0);
    glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE, 6 * sizeof(GLfloat), (void*)(3 * sizeof(GLfloat)));
    glEnableVertexAttribArray(1);
    glBindBuffer(GL_ARRAY_BUFFER, NULL);
    glBindVertexArray(NULL);
}
```

VBO: 顶点缓冲对象，存储顶点坐标

`glGenBuffers` 创建 Buffer

`glBindBuffer` 绑定指定类型的 Buffer （指向内存中我们创建的顶点、index 数组等）

使用 `glVertexAttribPointer` 可以设置顶点坐标的 layout （对应于 vertex shader 中的 layout）。设置好后使用 `glEnableVertexAttribArray` 启用。

EBO: 元素(index) 缓冲对象，存储顶点的绘制顺序。如果使用 EBO ，要用 `glDrawElements` ，否则使用 `glDrawArrays`

```cpp
// 使用 EBO
glDrawElements(GL_TRIANGLES, sizeof(indices), GL_UNSIGNED_INT, nullptr);
// 不使用 EBO
glDrawArrays(GL_TRIANGLES, 0, sizeof(vertices));
```

VAO: 顶点数组对象，可以将我们配置的 VBO、EBO “打包”起来，将来绘制的时候只需要 `glBindVertexArray` 即可。

在创建和绑定 VBO, EBO 的时候，先创建和绑定 VAO (`glBindVertexArray(VAO)`)。VBO 和 EBO 配置好后，需要调用 `glBindVertexArray(NULL)` 。


## glm

[g-truc/glm: OpenGL Mathematics (GLM)](https://github.com/g-truc/glm)

glm 可以进行向量、矩阵运算

### 向量

以 vec4 为例，向量的结构体中包含了四个 T 类型的成员，依次表示 x, y, z, w 四个分量。

```cpp
// Include/glm/detail/type_vec4.hpp
	template<typename T, qualifier Q>
	struct vec<4, T, Q>
	{
			union { T x, r, s; };
			union { T y, g, t; };
			union { T z, b, p; };
			union { T w, a, q; };
```

使用 `operator[]` 访问的时候是这样的：

> 成员函数的实现都放在类型对应的 `.inl` 文件里面，由于这些函数都是 `inline` 的，因此后缀名是 `.inl`

```cpp
// Include/glm/detail/type_vec4.inl
	template<typename T, qualifier Q>
	GLM_FUNC_QUALIFIER GLM_CONSTEXPR T& vec<4, T, Q>::operator[](typename vec<4, T, Q>::length_type i)
	{
		assert(i >= 0 && i < this->length());
		switch(i)
		{
		default:
		case 0:
			return x;
		case 1:
			return y;
		case 2:
			return z;
		case 3:
			return w;
		}
	}
```

### 矩阵

以 mat4 （4x4 矩阵）为例：

```cpp
// Include/glm/detail/type_mat4x4.hpp
	template<typename T, qualifier Q>
	struct mat<4, 4, T, Q>
	{
		typedef vec<4, T, Q> col_type;
	private:
		col_type value[4];
```

矩阵以列优先存储，也就是包含了一个指向列向量的指针。

矩阵相乘：

```cpp
// Include/glm/detail/type_mat4x4.inl
	template<typename T, qualifier Q>
	GLM_FUNC_QUALIFIER mat<4, 4, T, Q> operator*(mat<4, 4, T, Q> const& m1, mat<4, 4, T, Q> const& m2)
	{
		typename mat<4, 4, T, Q>::col_type const SrcA0 = m1[0];
		typename mat<4, 4, T, Q>::col_type const SrcA1 = m1[1];
		typename mat<4, 4, T, Q>::col_type const SrcA2 = m1[2];
		typename mat<4, 4, T, Q>::col_type const SrcA3 = m1[3];

		typename mat<4, 4, T, Q>::col_type const SrcB0 = m2[0];
		typename mat<4, 4, T, Q>::col_type const SrcB1 = m2[1];
		typename mat<4, 4, T, Q>::col_type const SrcB2 = m2[2];
		typename mat<4, 4, T, Q>::col_type const SrcB3 = m2[3];

		mat<4, 4, T, Q> Result;
		Result[0] = SrcA0 * SrcB0[0] + SrcA1 * SrcB0[1] + SrcA2 * SrcB0[2] + SrcA3 * SrcB0[3];
		Result[1] = SrcA0 * SrcB1[0] + SrcA1 * SrcB1[1] + SrcA2 * SrcB1[2] + SrcA3 * SrcB1[3];
		Result[2] = SrcA0 * SrcB2[0] + SrcA1 * SrcB2[1] + SrcA2 * SrcB2[2] + SrcA3 * SrcB2[3];
		Result[3] = SrcA0 * SrcB3[0] + SrcA1 * SrcB3[1] + SrcA2 * SrcB3[2] + SrcA3 * SrcB3[3];
		return Result;
	}
```

上面的代码等价于：

$$
A=
\left[
 \begin{matrix}
  A_0 & A_1 & A_2 & A_3
 \end{matrix}
\right]
\\
B=
\left[
 \begin{matrix}
  b_{00} & b_{01} & b_{02} & b_{03} \\
  b_{10} & b_{11} & b_{12} & b_{13} \\
  b_{20} & b_{21} & b_{22} & b_{23} \\
  b_{30} & b_{31} & b_{32} & b_{33} \\
 \end{matrix}
\right]
\\
A \cdot B =
\left[
 \begin{matrix}
  A_0b_{00} + A_1b_{10} + A_2b_{20} + A_3b_{30} &
  A_0b_{01} + A_1b_{11} + A_2b_{21} + A_3b_{31} &
  A_0b_{02} + A_1b_{12} + A_2b_{22} + A_3b_{32} &
  A_0b_{03} + A_1b_{13} + A_2b_{23} + A_3b_{33}
 \end{matrix}
\right]
$$

可见仅仅是存储为列优先形式，并不是将矩阵转置进行存储。

但是在访问矩阵的时候是先列后行，如 `a[2][1]` 表示 1 行 2 列。

## 3D

[LearnOpenGL-CN/08 Coordinate Systems.md at new-theme · LearnOpenGL-CN/LearnOpenGL-CN](https://github.com/LearnOpenGL-CN/LearnOpenGL-CN/blob/new-theme/docs/01%20Getting%20started/08%20Coordinate%20Systems.md)

[LearnOpenGL-CN/09 Camera.md at new-theme · LearnOpenGL-CN/LearnOpenGL-CN](https://github.com/LearnOpenGL-CN/LearnOpenGL-CN/blob/new-theme/docs/01%20Getting%20started/09%20Camera.md)

OpenGL 的 vertex shader 输出的是一个标准化设备坐标，`(x, y, z)` 在 `[-1.0, 1.0]` 范围，分别表示横坐标、纵坐标和深度，这是我们最终在窗口中看到的坐标。

我们要自己将 3D 空间的图形变换到这个坐标系上。

![](https://github.com/LearnOpenGL-CN/LearnOpenGL-CN/blob/new-theme/docs/img/01/08/coordinate_systems.png)

vertex shader:

```glsl
#version 330 core
layout (location = 0) in vec3 aPos;
layout (location = 1) in vec2 aTex;
out vec2 TexCoord;

uniform mat4 model;
uniform mat4 view;
uniform mat4 projection;

void main()
{
    gl_Position = projection * view * model * vec4(aPos, 1.0);
    TexCoord = aTex;
}
```

输入 aPos 是物体的局部坐标（Local Coordinate）

乘上 model 就变成在世界坐标（World Coordinate），即我们将物体的坐标变换到世界坐标系下，给它指定了位置、方向等。

再乘上 view 即观察坐标（View Coordinate），变换到了 Camera 的坐标系下。Camera 的 z 方向即观察的方向，xy 组成了投影面。

最后乘上 projection 得到裁剪坐标（Clip Coordinate），一般是一个投影变换矩阵，将 Camera 坐标系的三维坐标投影到投影面（xy 平面）上的 `[-1,1]*[-1,1]` 区域内。

### Camera

假设摄像机在世界坐标系下的某个位置 $P=(x_p,y_p,z_p)$ ，看向世界坐标系中的某一点 $P_0$ ，那么 $P_0-P$ 就是摄像机的 z 轴负方向，即正方向为 $P-P_0$ ，将其标准化（取单位向量），就得到 z 轴方向 $D$ 。

接着还要确定摄像机的 x、y 轴方向。我们取一个参考向量“上向量” $g$ ，一般是在世界坐标系中竖直向上（沿世界坐标系 y 轴）的向量，然后用 $g \times D$ 标准化，就得到 x 轴方向（右轴） $R$ ，这个方向是垂直于上向量的，比较符合我们的观察习惯。然后 y 轴的方向（上轴）就是 $U = D \times R$ 。容易证明，$R, U, D$ 构成右手系。

假设观察坐标系（即摄像机的坐标系）中有一个坐标 $(x', y', z')$，我们把它变换到世界坐标系中的 $(x, y, z)$ ，是这样的：

$$
\left[
 \begin{matrix}
  R & U & D
 \end{matrix}
\right]
\cdot
\left[
 \begin{matrix}
  x' \\
  y' \\
  z'
 \end{matrix}
\right] +
P =
\left[
 \begin{matrix}
  x \\
  y \\
  z
 \end{matrix}
\right]
$$

利用仿射（透视？）坐标，也可以写成四阶矩阵和四维向量的形式：

$$
\left[
 \begin{matrix}
  I_3 & P \\
  0 & 1
 \end{matrix}
\right]
\cdot
\left[
 \begin{matrix}
  R & U & D & 0 \\
  0 & 0 & 0 & 1
 \end{matrix}
\right]
\cdot
\left[
 \begin{matrix}
  x' \\
  y' \\
  z' \\
  1
 \end{matrix}
\right] =
\left[
 \begin{matrix}
  x \\
  y \\
  z \\
  1
 \end{matrix}
\right]
$$

左边就是观察坐标系到世界坐标系的变换矩阵。

由此，我们也可以得到世界坐标系到观察坐标系的反变换矩阵。由于 $R, U, D$ 是正交单位向量，其逆矩阵很容易得出，就是转置矩阵。

$$
\left[
 \begin{matrix}
  R^T & 0 \\
  U^T & 0 \\
  D^T & 0 \\
  0 & 1
 \end{matrix}
\right]
\cdot
\left[
 \begin{matrix}
  I_3 & -P \\
  0 & 1
 \end{matrix}
\right]
\cdot
\left[
 \begin{matrix}
  x \\
  y \\
  z \\
  1
 \end{matrix}
\right] =
\left[
 \begin{matrix}
  x' \\
  y' \\
  z' \\
  1
 \end{matrix}
\right]
$$

因此我们可以得到用于 view 的 lookAt 矩阵。

$$
\left[
 \begin{matrix}
  R^T & -R \cdot P \\
  U^T & -U \cdot P \\
  D^T & -D \cdot P \\
  0 & 1
 \end{matrix}
\right]
\cdot
$$

我们看看 glm 中的实现：

```cpp
// Include/glm/ext/matrix_transform.inl
	template<typename T, qualifier Q>
	GLM_FUNC_QUALIFIER mat<4, 4, T, Q> lookAtRH(vec<3, T, Q> const& eye, vec<3, T, Q> const& center, vec<3, T, Q> const& up)
	{
		vec<3, T, Q> const f(normalize(center - eye));
		vec<3, T, Q> const s(normalize(cross(f, up)));
		vec<3, T, Q> const u(cross(s, f));

		mat<4, 4, T, Q> Result(1);
		Result[0][0] = s.x;
		Result[1][0] = s.y;
		Result[2][0] = s.z;
		Result[0][1] = u.x;
		Result[1][1] = u.y;
		Result[2][1] = u.z;
		Result[0][2] =-f.x;
		Result[1][2] =-f.y;
		Result[2][2] =-f.z;
		Result[3][0] =-dot(s, eye);
		Result[3][1] =-dot(u, eye);
		Result[3][2] = dot(f, eye);
		return Result;
	}
```

glm 中，我们只要提供摄像机坐标、观察点坐标和上向量就可以构造 lookAt 矩阵（此处是生成右手坐标系的）。

可见，`eye` 就是位置 $P$ ， `center` 是观察目标点， `f = norm(center-eye)` 就是 $-D$ 。`up` 就是上向量 $g$ 。

`s, u` 就是 $R, U$ 。由于 `f` 是 $-D$ ，因此叉乘的时候交换了次序。

## projection

[OpenGL Projection Matrix](http://www.songho.ca/opengl/gl_projectionmatrix.html)

# sphere

[OpenGL Sphere](http://www.songho.ca/opengl/gl_sphere.html)
