# 1. URP とは

RP = Render pipelines とは、Scene の contents をディスプレイのスクリーン上に、表示するの一連の流れのこと。

## 一連の流れとは?
一連の流れは、３つの主要な段階で決定される。
- Culling
- Rendering
- Post-processing

これらの用語の定義については後に記載する。

## 異なるレンダーパイプライン
- Built-in Render Pipeline
  - Unity's default render pipelien
- Universal Render Pipeline
  - This is a Scriptabl render pipeline
  - 様々なプラットフォーム対応
- High Definition Render Pipeline
  - This is a Scrptable render pipeline
  - is for high-end platforms


## Definitions
いくつかの重要な graphics rendering terms について確認していく

- Render Pipeline
a render pipeline determines how the objects in your scene are displayed, in there main steges.

- Culling
The first step is  culling; it lists the objects that need to be rendered, preferable  the ones visible to the camera(frustum culling) and unoccluded by other objects(occlusion culling)

**Redering(描画)するObjectsを決定**

- Rendering
The second stage, rendering, is the drawing of the these objects, with the correct lighting and some of their properties, into pixel-based buffers.

**この時点では、Pixed buffers に書き込んでいるだけ!!**

- Post Processing
Finally, post-processing operations can be carried out on these buffers, for instance applying color grading, bloom and depth of field, to generate the final output frame that is sent to a display screen

**ここで、最終的なアウトプットが決定!!**


These operations are repeated many times a second, depending on the frame rate.


~~~: Render pipline stages
1. Culling : List objects to render
2. Rendering : Draw Objects
3. Post Processing: Apply additional image effects
~~~

ここまでが、大まかなレンダリンングするまでの工程の説明である。以下には、各工程で、Shader・Light などの処理が、どこの過程で、どのように実行されるのかが記載される。

#### Shader
- Shader
A Shader is a generic name for a program, or a collection of programs, running on the Graphics Processing Unit (GPU). For instance, after the culling stage is completed, a Vertex Shader
 is used to transform the vertex coordinates of the visible objects from “object space” into a different space called “clip space”; these new coordinates are then used by the GPU to rasterize the scene, i.e. convert the vectorial representation of the scene into actual pixels. At a later stage, these pixels will be colored by pixel
 (or fragment) shaders; the pixel color will generally depend on the material properties of the respective surface and the surrounding lighting. Another common type of shader available on modern hardware is Compute Shaders: they allow programmers to exploit the considerable parallel processing power of GPUs for any kind of mathematical operations, such as light culling, particle physics, or volumetric simulation.

- Object Space
3Dモデル自身を中心とした座標系。モデルの原点(0,0,0)を基準とする座標のこと。モデル作成時の座標系であり、オブジェクトの形状を定義する。これは、World内での位置・回転・スケールとは独立している(異なる)。

~~~:hlsl (Vertex Shaderの入力)
struct Attributes
{
    float4 positionOS : POSITION;  // OS = Object Space
    float3 normalOS   : NORMAL;    // 法線もObject Space
    float2 uv         : TEXCOORD0;
};
~~~

- Clip Space
カメラの視錐台（View Frustum）を正規化立方体 [-1,1] に変換した座標系

~~~: 座標変換の流れ
Object Space 
→ World Space 
→ View Space  
→ Clip Space
~~~

~~~:hlsl
// 1. Object Space → World Space
float4 positionWS = mul(unity_ObjectToWorld, positionOS);

// 2. World Space → View Space  
float4 positionVS = mul(UNITY_MATRIX_V, positionWS);

// 3. View Space → Clip Space
float4 positionCS = mul(UNITY_MATRIX_P, positionVS);

// または一括変換
float4 positionCS = TransformObjectToHClip(positionOS.xyz);
~~~

>[クリップスペース](https://www.reddit.com/r/Unity3D/comments/8u876t/can_you_eli5_what_is_clip_space_position/?tl=ja)
は、メッシュ内の頂点を画面上のピクセル座標に変換する過程の一段階です。
3Dオブジェクトをモデル化する際、モデルの原点（0,0,0）を基準とした頂点（位置）を作成します。レンダリングする際、これらの位置は最初に世界の原点を基準に変換されます。次に、世界内の位置はカメラを基準に変換されます。たとえば、カメラの真前、距離1の点は（0, 0, 1）になります。カメラのビューには特定の形状（フラスタム）があり、これがレンダリングされた画像における「パースペクティブ効果」の強さと、シーンのどのくらいが見えるかを決定します。カメラを基準とした位置は、この効果を生み出すためにフラスタムの形状に合わせて変換/歪められます。この時点で、画面に表示されるすべての頂点は、-1.0から+1.0の範囲の座標を持っています（X軸とY軸の両方）。これが「クリップスペース」と呼ばれるものです。クリップスペースと呼ばれるのは、この時点でGPUが画面に（完全に）表示されない形状を切り取ることができるからです。

- Rasterize
ベクトル形式のデータを、ピクセル（画素）に変換する処理
~~~: 処理フロー
Clip Space座標 
→ Screen Space座標 
→ ピクセル生成 
→ Fragment Shader実行
~~~


----

## Buffer とは?
Buffer(バッファ)は、IT分野では、データを一時的に蓄える記憶領域のことを指す。
- Buffers
レンダリング処理の各段階で、ピクセル単位の情報を格納する2次元の配列（メモリ領域)のこと。

最終的なアウトプットである、画面上に映し出す 各 pixel をどのように表示するかを決定するために、Buffer とうい各ピクセルの座標位置毎に情報を格納した ２次元配列あ流。２次元配列は Color・Depth などUnityにおける画面に映し出す際に必要な様々なことを考慮した情報がそれぞれ格納される。その情報、複数のBufferが連携した内容が、 各pixelが、画面上に表示される。

レンダリング処理の結果をBufferに保存 → Bufferの内容が最終的な色になるのである。

~~~
1. 3Dオブジェクト情報
   ↓
2. 各種Buffer（2次元配列）に中間結果を保存
   ↓  
3. Bufferの情報を組み合わせて最終的なpixel色を決定
   ↓
4. 画面に表示
~~~

### 主な Rendering Buffer の種類
- Color Buffer
- Depth Buffer
- Stencil Buffer など

### Buffer 処理における、一連の流れ
Forward・Deferred Rendering と Rendering Path の違いによって、実際のフローは変わる

~~~ : Forward Rendering の例
A[3D Objects] --> B[Vertex Shader]
B --> C[Rasterization]
C --> D[Fragment Shader]
D --> E[Depth Test]
E --> F[Color Buffer]
F --> G[Post-Processing]
G --> H[Final Frame]
~~~

### 具体例
~~~: イメージ
具体例で考えてみましょう
画面の座標 (100, 200) のピクセルを表示する場合：
Color Buffer[100][200]    = RGB(255, 128, 64)  // オレンジ色
Depth Buffer[100][200]    = 0.8                // カメラから80%の距離
Stencil Buffer[100][200]  = 1                  // 特殊効果フラグ
~~~
この情報をもとに、最終的にそのピクセルが：
オレンジ色で表示される
手前にある別のオブジェクトがあれば隠される（Depth値による判定）
特殊効果が適用される（Stencil値による判定）


---
# 参考
[レンダーパイプラインとライティングソリューションの選択と設定 - Unity マニュアル](https://docs.unity3d.com/ja/2023.2/Manual/BestPracticeLightingPipelines.html)

[Reddit - クリップスペースについて](https://www.reddit.com/r/Unity3D/comments/8u876t/can_you_eli5_what_is_clip_space_position/?tl=ja)

[Unity Rendering Pipeline Concepts \| Claude](https://claude.ai/share/74edef5c-2e26-4450-8081-1780d14a4090)


---
# 次にやること

しかし、正直なところ、クリップスペースが何であるかを理解する必要がある場合は、3Dレンダリングパイプラインについてもっと詳しく知っておく必要があります。私がこの分野に入り始めた頃、次の2つの記事が役に立ちました。これらは、一般的なアイデアを与え、たくさんの役立つ画像があるはずです。（そして、このサイトの他のほとんどすべて。本当に素晴らしいリソースです）
（Unityに特化したものではありませんが、同じ概念が適用されます）

以下の記事を理解する！

[opengl-tutorial](http://www.opengl-tutorial.org/beginners-tutorials/tutorial-3-matrices/)

[Computing the Pixel Coordinates of a 3D Point](http://www.scratchapixel.com/lessons/3d-basic-rendering/computing-pixel-coordinates-of-3d-point/mathematics-computing-2d-coordinates-of-3d-points) 

