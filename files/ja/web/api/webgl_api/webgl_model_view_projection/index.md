---
title: WebGL モデル ビュー 射影
slug: Web/API/WebGL_API/WebGL_model_view_projection
translation_of: Web/API/WebGL_API/WebGL_model_view_projection
---
<p>{{WebGLSidebar}}</p>

<p class="summary">この記事では、<a href="/ja/docs/Web/API/WebGL_API">WebGL</a> プロジェクト内でデータを取得し、それを適切な空間に投影して画面に表示する方法について説明します。並進、拡縮、回転行列を使用した基本的な行列計算の知識があることを前提としています。3Dシーンを構成するときに通常使用される中心的な3つの行列である、モデル、ビュー、射影行列について説明します。</p>

<div class="note">
<p><strong>Note</strong>: This article is also available as an <a href="https://github.com/TatumCreative/mdn-model-view-projection">MDN content kit</a>. It also uses a collection of <a href="https://github.com/TatumCreative/mdn-webgl">utility functions</a> available under the <code>MDN</code> global object.</p>
</div>

<h2 id="モデル、ビュー、射影行列">モデル、ビュー、射影行列</h2>

<p>WebGLの空間内の点とポリゴンの個々の変換は、並進、拡縮、回転などの基本的な変換行列によって処理されます。 <span class="tlid-translation translation" lang="ja"><span title="">これらの行列は、複雑な3Dシーンの描画に役立つように、一緒に構成し、特別な方法でグループ化できます。</span></span>これらの構成された行列は、最終的に元のモデルデータを<strong>クリップ空間</strong>と呼ばれる特別な座標空間に移動します。これは2ユニットの幅の立方体で、中心が (0,0,0) 、対角が (-1,-1,-1) から (1,1,1) になります。このクリップ空間は2次元平面に圧縮され、画像へラスタライズされます。</p>

<p><span class="tlid-translation translation" lang="ja"><span title="">以下で説明する最初の行列は<strong>モデル行列</strong>です。これは、元のモデルデータを取得して3次元ワールド空間内で移動する方法を定義します。</span></span> <span class="tlid-translation translation" lang="ja"><span title=""><strong>射影行列</strong>は、ワールド空間座標をクリップ空間座標に変換するために使用されます。</span></span> <span class="tlid-translation translation" lang="ja"><span title="">一般的に使用される射影行列である<strong>透視投影射影行列</strong>は、3D仮想世界の視聴者の代理として機能する一般的なカメラの<em>効果</em>を模倣するために使用されます。</span></span> <span class="tlid-translation translation" lang="ja"><span title=""><strong>ビュー行列</strong>は、変更されるカメラの位置をシミュレートし、シーン内のオブジェクトを移動して視聴者が現在何を見られるかを変更します</span></span>。</p>

<p><span class="tlid-translation translation" lang="ja"><span title="">以下のセクションでは、モデル、ビュー、射影行列の背景にある考え方と実装について詳説します。</span> <span title="">これらの行列は、画面上でデータを移動するための根幹であり、個々のフレームワークやエンジンを超える概念です。</span></span></p>

<h2 id="クリップ空間">クリップ空間</h2>

<p>WebGLプログラムでは、通常、データは自分の座標系でGPUにアップロードされ、次に頂点シェーダーがそれらの点を<strong>クリップ空間</strong>と呼ばれる特別な座標系に変換します。クリップ空間の外側にあるデータは切り取られ、描画されません。ただし、三角形がこのスペースの境界を跨ぐ場合は、新しい三角形に分割され、クリップスペースにある新しい三角形の部分のみが残ります。</p>

<p><img alt="A 3d graph showing clip space in WebGL." src="https://mdn.mozillademos.org/files/11371/clip-space-graph.svg" style="height: 432px; width: 500px;"></p>

<p>上の図は、全ての点が収まる必要のあるクリップ空間を視覚化したものです。これは、各辺が2の立方体であり、片方の角が (-1,-1,-1) にあり、対角が (1,1,1) にあります。立方体の中心は点 (0,0,0) です。 クリップ空間に使用されるこの8立方メートルの座標系は、正規化デバイス座標（NDC）と呼ばれます。WebGLコードを調べて作業している間、その用語を時々耳にするかもしれません。</p>

<p><span class="tlid-translation translation" lang="ja"><span title="">このセクションでは、データをクリップ空間座標系に直接配置する仕組みを説明します。</span> <span title="">通常、任意の座標系にあるモデルデータが使用され、その後、行列を使用して変換され、モデル座標がクリップ空間座標系に変換されます。この例では、クリップ空間がどのように機能するかを最も簡単に説明する為、単純に (-1, -1, -1) から (1,1,1) までの範囲のモデル座標を使用します。</span><span title="">以下のコードは、画面上に正方形を描く為に2つの三角形を作成します。</span><span title="">正方形のZ深度は、正方形が同じ空間を共有するときに何が上に描画されるかを決定します。</span><span title="">小さいZ値は大きいZ値の上にレンダリングされます。</span></span></p>

<h3 id="WebGLBox_の例">WebGLBox の例</h3>

<p><span class="tlid-translation translation" lang="ja">この例では、画面上に2Dボックスを描画するカスタム</span> <code>WebGLBox</code> <span class="tlid-translation translation" lang="ja">オブジェクトを作成します</span>。</p>

<div class="note">
<p><strong>Note</strong>: The code for each WebGLBox example is available in this <a href="https://github.com/TatumCreative/mdn-model-view-projection/tree/master/lessons">github repo</a> and is organized by section. In addition there is a JSFiddle link at the bottom of each section.</p>
</div>

<h4 id="WebGLBox_コンストラクタ">WebGLBox コンストラクタ</h4>

<p><span class="tlid-translation translation" lang="ja">コンストラクターは次のようになります。</span></p>

<pre class="brush: js notranslate">function WebGLBox() {
  // Setup the canvas and WebGL context
  this.canvas = document.getElementById('canvas');
  this.canvas.width = window.innerWidth;
  this.canvas.height = window.innerHeight;
  this.gl = MDN.createContext(canvas);

  var gl = this.gl;

  // Setup a WebGL program, anything part of the MDN object is defined outside of this article
  this.webglProgram = MDN.createWebGLProgramFromIds(gl, 'vertex-shader', 'fragment-shader');
  gl.useProgram(this.webglProgram);

  // Save the attribute and uniform locations
  this.positionLocation = gl.getAttribLocation(this.webglProgram, 'position');
  this.colorLocation = gl.getUniformLocation(this.webglProgram, 'color');

  // Tell WebGL to test the depth when drawing, so if a square is behind
  // another square it won't be drawn
  gl.enable(gl.DEPTH_TEST);

}
</pre>

<h4 id="WebGLBox_描画">WebGLBox 描画</h4>

<p><span class="tlid-translation translation" lang="ja">次に、画面上にボックスを描画するメソッドを作成します</span>。</p>

<pre class="brush: js notranslate">WebGLBox.prototype.draw = function(settings) {
  // Create some attribute data; these are the triangles that will end being
  // drawn to the screen. There are two that form a square.

  var data = new Float32Array([

    //Triangle 1
    settings.left,  settings.bottom, settings.depth,
    settings.right, settings.bottom, settings.depth,
    settings.left,  settings.top,    settings.depth,

    //Triangle 2
    settings.left,  settings.top,    settings.depth,
    settings.right, settings.bottom, settings.depth,
    settings.right, settings.top,    settings.depth
  ]);

  // Use WebGL to draw this onto the screen.

  // Performance Note: Creating a new array buffer for every draw call is slow.
  // This function is for illustration purposes only.

  var gl = this.gl;

  // Create a buffer and bind the data
  var buffer = gl.createBuffer();
  gl.bindBuffer(gl.ARRAY_BUFFER, buffer);
  gl.bufferData(gl.ARRAY_BUFFER, data, gl.STATIC_DRAW);

  // Setup the pointer to our attribute data (the triangles)
  gl.enableVertexAttribArray(this.positionLocation);
  gl.vertexAttribPointer(this.positionLocation, 3, gl.FLOAT, false, 0, 0);

  // Setup the color uniform that will be shared across all triangles
  gl.uniform4fv(this.colorLocation, settings.color);

  // Draw the triangles to the screen
  gl.drawArrays(gl.TRIANGLES, 0, 6);
}
</pre>

<p><span class="tlid-translation translation" lang="ja">シェーダーは GLSL で記述されたコードの一部であり、データポイントを取得して最終的に画面に描画します。便宜上、これらのシェーダーは、カスタム関数 </span><code>MDN.createWebGLProgramFromIds()</code> <span class="tlid-translation translation" lang="ja">を介してプログラムに取り込まれる要素</span> {{htmlelement("script")}} <span class="tlid-translation translation" lang="ja">に格納されます。この関数は、これらのチュートリアル用に作成された</span> <a href="https://github.com/TatumCreative/mdn-webgl">ユーティリティ関数群</a> <span class="tlid-translation translation" lang="ja">の一部であり、ここでは詳しく説明しません。この関数は、いくつかの GLSL ソースコードを取得して WebGL プログラムにコンパイルする基本を処理します。関数は3つのパラメーターを取ります。プロ</span>グ<span class="tlid-translation translation" lang="ja">ラムをレンダリングするコンテキスト、頂点シェーダーを含む要素のID </span>{{htmlelement("script")}}<span class="tlid-translation translation" lang="ja">、フラグメントシェーダーを含む要素のID </span>{{htmlelement("script")}} <span class="tlid-translation translation" lang="ja">です。頂点シェーダーは頂点を配置し、フラグメントシェーダーは各ピクセルに色を付けます。</span></p>

<p><span class="tlid-translation translation" lang="ja">最初に、</span><span class="tlid-translation translation" lang="ja"><span class="alt-edited" title="">画面上で頂点を移動させる頂点シェーダーを見てみましょう。</span></span></p>

<pre class="brush: glsl notranslate">// The individual position vertex
attribute vec3 position;

void main() {
  // the gl_Position is the final position in clip space after the vertex shader modifies it
  gl_Position = vec4(position, 1.0);
}
</pre>

<p><span class="tlid-translation translation" lang="ja"><span title="">次に、データを実際にピクセルにラスタライズするために、フラグメントシェーダーはピクセルごとに全てを評価し、一つの色を設定します。</span><span title="">GPUは、レンダリングする必要があるピクセルごとにシェーダー関数を呼び出します。</span><span title="">シェーダーの仕事は、そのピクセルに使用する色を返すことです。</span></span></p>

<pre class="brush: glsl notranslate">precision mediump float;
uniform vec4 color;

void main() {
  gl_FragColor = color;
}
</pre>

<p><span class="tlid-translation translation" lang="ja"><span title="">これらの設定が含まれているので、クリップ空間座標を使用して画面に直接描画します。</span></span></p>

<pre class="brush: js notranslate">var box = new WebGLBox();
</pre>

<p><span class="tlid-translation translation" lang="ja"><span title="">最初に中央に赤いボックスを描きます。</span></span></p>

<pre class="brush: js notranslate">box.draw({
  top    : 0.5,             // x
  bottom : -0.5,            // x
  left   : -0.5,            // y
  right  : 0.5,             // y

  depth  : 0,               // z
  color  : [1, 0.4, 0.4, 1] // red
});
</pre>

<p>次に、緑色のボックスを赤いボックスの上部に描画します。</p>

<pre class="brush: js notranslate">box.draw({
  top    : 0.9,             // x
  bottom : 0,               // x
  left   : -0.9,            // y
  right  : 0.9,             // y

  depth  : 0.5,             // z
  color  : [0.4, 1, 0.4, 1] // green
});
</pre>

<p>最後に、クリッピングが実際に行われていることを示すために、このボックスは完全にクリップ空間の外側にあるため、描画されません。深さが -1.0 から 1.0 の範囲外です。</p>

<pre class="brush: js notranslate">box.draw({
  top    : 1,               // x
  bottom : -1,              // x
  left   : -1,              // y
  right  : 1,               // y

  depth  : -1.5,            // z
  color  : [0.4, 0.4, 1, 1] // blue
});
</pre>

<h4 id="結果">結果</h4>

<p><a href="https://jsfiddle.net/mff99yu5">JSFiddle で表示</a></p>

<p><img alt="The results of drawing to clip space using WebGL." src="https://mdn.mozillademos.org/files/11373/part1.png" style="height: 530px; width: 800px;"></p>

<h4 id="演習">演習</h4>

<p><span class="tlid-translation translation" lang="ja"><span title="">この時点で役立つ演習は、コードを変更してボックスをクリップ空間内で移動し、点がクリップ空間内でどのようにクリップされ、移動されるかを感じ取ることです。</span><span title="">背景を持つボックス状のスマイルのような絵を描いてみてください。</span></span></p>

<h2 id="斉次座標"><span class="ILfuVd">斉<strong>次座標</strong></span></h2>

<p>前のクリップ空間の頂点シェーダのメインラインは、このコードを含んでいました。</p>

<pre class="brush: js notranslate">gl_Position = vec4(position, 1.0);
</pre>

<p><span class="tlid-translation translation" lang="ja"><span title="">変数</span></span> <code>position</code> <span class="tlid-translation translation" lang="ja"><span title="">は、</span></span> <code>draw()</code> <span class="tlid-translation translation" lang="ja"><span title="">メソッドで定義され、属性としてシェーダーに渡されました。</span><span title="">これは3次元の点ですが、パイプラインを介して渡されることになる変数</span></span> <code>gl_Position</code> <span class="tlid-translation translation" lang="ja"><span title="">は実際には4次元です</span></span>。すなわち、 <code>(x, y, z)</code> <span class="tlid-translation translation" lang="ja"><span title="">の代わりに</span></span> <code>(x, y, z, w)</code><span class="tlid-translation translation" lang="ja"><span title=""> となっています</span></span><span class="tlid-translation translation" lang="ja"><span title="">。</span></span><code>z</code> <span class="tlid-translation translation" lang="ja"><span title="">の後には文字がないため、慣例により、この4番目の次元には</span></span> <code>w</code> <span class="tlid-translation translation" lang="ja"><span title="">というラベルが付いています。</span> <span title="">上記の例では、</span></span> <code>w</code> <span class="tlid-translation translation" lang="ja"><span title="">座標は 1.0 に設定されています。</span></span></p>

<p><span class="tlid-translation translation" lang="ja"><span title="">明らかな疑問は、「なぜ余分な次元があるのか？」です。</span><span title="">この追加により、3Dデータを操作するための多くの優れた手法が可能になることが分かります。</span><span title="">この追加された次元により、遠近法の概念が座標系に導入されます。</span><span title="">それを配置すると、3D座標を2D空間にマッピングできます。これにより、2本の平行線が遠くに離れるときに交差するようになります。</span><span title="">値</span></span> <code>w</code> <span class="tlid-translation translation" lang="ja"><span title="">は、座標の他のコンポーネントの除数として使用されるため、</span></span> <code>x</code><span class="tlid-translation translation" lang="ja"><span title="">、</span></span><code>y</code><span class="tlid-translation translation" lang="ja"><span title="">、</span></span><code>z</code><span class="tlid-translation translation" lang="ja"><span title="">の真の値は、</span></span><code>x/w</code><span class="tlid-translation translation" lang="ja"><span title="">、</span></span><code>y/w</code><span class="tlid-translation translation" lang="ja"><span title="">、</span></span><code>z/w</code><span class="tlid-translation translation" lang="ja"><span title="">として計算されます（そして、</span></span><code>w</code><span class="tlid-translation translation" lang="ja"><span title="">も</span> </span><code>w/w</code><span class="tlid-translation translation" lang="ja"><span title="">で1になる）。</span></span></p>

<p>A three dimensional point is defined in a typical Cartesian coordinate system. The added fourth dimension changes this point into a {{interwiki("wikipedia", "homogeneous coordinates", "homogeneous coordinate")}}. It still represents a point in 3D space and it can easily be demonstrated how to construct this type of coordinate through a pair of simple functions.</p>

<pre class="brush: js notranslate">function cartesianToHomogeneous(point)
  let x = point[0];
  let y = point[1];
  let z = point[2];

  return [x, y, z, 1];
}

function homogeneousToCartesian(point) {
  let x = point[0];
  let y = point[1];
  let z = point[2];
  let w = point[3];

  return [x/w, y/w, z/w];
}
</pre>

<p>As previously mentioned and shown in the functions above, the w component divides the x, y, and z components. When the w component is a non-zero real number then homogeneous coordinate easily translates back into a normal point in Cartesian space. Now what happens if the w component is zero? In JavaScript the value returned would be as follows.</p>

<pre class="brush: js notranslate">homogeneousToCartesian([10, 4, 5, 0]);
</pre>

<p>This evaluates to: <code>[Infinity, Infinity, Infinity]</code>.</p>

<p>This homogeneous coordinate represents some point at infinity. This is a handy way to represent a ray shooting off from the origin in a specific direction. In addition to a ray, it could also be thought of as a representation of a directional vector. If this homogeneous coordinate is multiplied against a matrix with a translation then the translation is effectively stripped out.</p>

<p>When numbers are extremely large (or extremely small) on computers they begin to become less and less precise because there are only so many ones and zeros that are used to represent them. The more operations that are done on larger numbers, the more and more errors accumulate into the result. When dividing by w, this can effectively increase the precision of very large numbers by operating on two potentially smaller, less error-prone numbers.</p>

<p>The final benefit of using homogeneous coordinates is that they fit very nicely for multiplying against 4x4 matrices. A vertex must match at least one of the dimensions of a matrix in order to be multiplied against it. The 4x4 matrix can be used to encode a variety of useful transformations. In fact, the typical perspective projection matrix uses the division by the w component to achieve its transformation.</p>

<p>The clipping of points and polygons from clip space actually happens after the homogeneous coordinates have been transformed back into Cartesian coordinates (by dividing by w). This final space is known as <strong>normalized device coordinates</strong> or NDC.</p>

<p>To start playing with this idea the previous example can be modified to allow for the use of the <code>w</code> component.</p>

<pre class="brush: js notranslate">//Redefine the triangles to use the W component
var data = new Float32Array([
  //Triangle 1
  settings.left,  settings.bottom, settings.depth, settings.w,
  settings.right, settings.bottom, settings.depth, settings.w,
  settings.left,  settings.top,    settings.depth, settings.w,

  //Triangle 2
  settings.left,  settings.top,    settings.depth, settings.w,
  settings.right, settings.bottom, settings.depth, settings.w,
  settings.right, settings.top,    settings.depth, settings.w
]);
</pre>

<p>Then the vertex shader uses the 4 dimensional point passed in.</p>

<pre class="brush: js notranslate">attribute vec4 position;

void main() {
  gl_Position = position;
}
</pre>

<p>First, we draw a red box in the middle, but set W to 0.7. As the coordinates get divided by 0.7 they will all be enlarged.</p>

<pre class="brush: js notranslate">box.draw({
  top    : 0.5,             // y
  bottom : -0.5,            // y
  left   : -0.5,            // x
  right  : 0.5,             // x
  w      : 0.7,             // w - enlarge this box

  depth  : 0,               // z
  color  : [1, 0.4, 0.4, 1] // red
});
</pre>

<p>Now, we draw a green box up top, but shrink it by setting the w component to 1.1</p>

<pre class="brush: js notranslate">box.draw({
  top    : 0.9,             // y
  bottom : 0,               // y
  left   : -0.9,            // x
  right  : 0.9,             // x
  w      : 1.1,             // w - shrink this box

  depth  : 0.5,             // z
  color  : [0.4, 1, 0.4, 1] // green
});
</pre>

<p>This last box doesn't get drawn because it's outside of clip space. The depth is outside of the -1.0 to 1.0 range.</p>

<pre class="brush: js notranslate">box.draw({
  top    : 1,               // y
  bottom : -1,              // y
  left   : -1,              // x
  right  : 1,               // x
  w      : 1.5,             // w - Bring this box into range

  depth  : -1.5,             // z
  color  : [0.4, 0.4, 1, 1] // blue
});
</pre>

<h3 id="The_results">The results</h3>

<p><a href="https://jsfiddle.net/mff99yu">View on JSFiddle</a></p>

<p id="sect1"><img alt="The results of using homogeneous coordinates to move the boxes around in WebGL." src="https://mdn.mozillademos.org/files/11375/part2.png" style="height: 530px; width: 800px;"></p>

<h3 id="Exercises">Exercises</h3>

<ul>
 <li>Play around with these values to see how it affects what is rendered on the screen. Note how the previously clipped blue box is brought back into range by setting its w component.</li>
 <li>Try creating a new box that is outside of clip space and bring it back in by dividing by w.</li>
</ul>

<h2 id="Model_transform">Model transform</h2>

<p>Placing points directly into clip space is of limited use. In real world applications, you don't have all your source coordinates already in clip space coordinates. So most of the time, you need to transform the model data and other coordinates into clip space. The humble cube is an easy example of how to do this. Cube data consists of vertex positions, the colors of the faces of the cube, and the order of the vertex positions that make up the individual polygons (in groups of 3 vertices to construct the triangles composing the cube's faces). The positions and colors are stored in GL buffers, sent to the shader as attributes, and then operated upon individually.</p>

<p>Finally a single model matrix is computed and set. This matrix represents the transformations to be performed on every point making up the model in order to move it into the correct space, and to perform any other needed transforms on each point in the model. This applies not just to each vertex, but to every single point on every surface of the model as well.</p>

<p>In this case, for every frame of the animation a series of scale, rotation, and translation matrices move the data into the desired spot in clip space. The cube is the size of clip space (-1,-1,-1) to (1,1,1) so it will need to be shrunk down in order to not fill the entirety of clip space. This matrix is sent directly to the shader, having been multiplied in JavaScript beforehand.</p>

<p>The following code sample defines a method on the <code>CubeDemo</code> object that will create the model matrix. It uses custom functions to create and multiply matrices as defined in the <a href="https://github.com/TatumCreative/mdn-webgl">MDN WebGL</a> shared code. The new function looks like this:</p>

<pre class="brush: js notranslate">CubeDemo.prototype.computeModelMatrix = function(now) {
  //Scale down by 50%
  var scale = MDN.scaleMatrix(0.5, 0.5, 0.5);

  // Rotate a slight tilt
  var rotateX = MDN.rotateXMatrix(now * 0.0003);

  // Rotate according to time
  var rotateY = MDN.rotateYMatrix(now * 0.0005);

  // Move slightly down
  var position = MDN.translateMatrix(0, -0.1, 0);

  // Multiply together, make sure and read them in opposite order
  this.transforms.model = MDN.multiplyArrayOfMatrices([
    position, // step 4
    rotateY,  // step 3
    rotateX,  // step 2
    scale     // step 1
  ]);
};
</pre>

<p>In order to use this in the shader it must be set to a uniform location. The locations for the uniforms are saved in the <code>locations</code> object shown below:</p>

<pre class="brush: js notranslate">this.locations.model = gl.getUniformLocation(webglProgram, 'model');
</pre>

<p>And finally the uniform is set to that location. This hands off the matrix to the GPU.</p>

<pre class="brush: js notranslate">gl.uniformMatrix4fv(this.locations.model, false, new Float32Array(this.transforms.model));
</pre>

<p>In the shader, each position vertex is first transformed into a homogeneous coordinate (a <code>vec4</code> object), and then multiplied against the model matrix.</p>

<pre class="brush: glsl notranslate">gl_Position = model * vec4(position, 1.0);
</pre>

<div class="note">
<p><strong>Note</strong>: In JavaScript, matrix multiplication requires a custom function, while in the shader it is built into the language with the simple * operator.</p>
</div>

<h3 id="The_results_2">The results</h3>

<p><a href="https://jsfiddle.net/5jofzgsh">View on JSFiddle</a></p>

<p><img alt="Using a model matrix" src="https://mdn.mozillademos.org/files/11377/part3.png" style="height: 530px; width: 800px;"></p>

<p>At this point the w value of the transformed point is still 1.0. The cube still doesn't have any perspective. The next section will take this setup and modify the w values to provide some perspective.</p>

<h3 id="Exercises_2">Exercises</h3>

<ul>
 <li>Shrink down the box using the scale matrix and position it in different places within clip space.</li>
 <li>Try moving it outside of clip space.</li>
 <li>Resize the window and watch as the box skews out of shape.</li>
 <li>Add a <code>rotateZ</code> matrix.</li>
</ul>

<h2 id="Divide_by_W">Divide by W</h2>

<p>An easy way to start getting some perspective on our model of the cube is to take the Z coordinate and copy it over to the w coordinate. Normally when converting a cartesian point to homogeneous it becomes <code>(x,y,z,1)</code>, but we're going to set it to something like <code>(x,y,z,z)</code>. In reality we want to make sure that z is greater than 0 for points in view, so we'll modify it slightly by changing the value to <code>((1.0 + z) * scaleFactor)</code>. This will take a point that is normally in clip space (-1 to 1) and move it into a space more like (0 to 1) depending on what the scale factor is set to. The scale factor changes the final w value to be either higher or lower overall.</p>

<p>The shader code looks like this.</p>

<pre class="brush: js notranslate">// First transform the point
vec4 transformedPosition = model * vec4(position, 1.0);

// How much effect does the perspective have?
float scaleFactor = 0.5;

// Set w by taking the z value which is typically ranged -1 to 1, then scale
// it to be from 0 to some number, in this case 0-1.
float w = (1.0 + transformedPosition.z) * scaleFactor;

// Save the new gl_Position with the custom w component
gl_Position = vec4(transformedPosition.xyz, w);
</pre>

<h3 id="The_results_3">The results</h3>

<p><a href="https://jsfiddle.net/vk9r8h2c">View on JSFiddle</a></p>

<p><img alt="Filling the W component and creating some projection." src="https://mdn.mozillademos.org/files/11379/part4.png" style="height: 531px; width: 800px;"></p>

<p>See that small dark blue triangle? That's an additional face added to our object because the rotation of our shape has caused that corner to extend outside clip space, thus causing the corner to be clipped away. See <a href="#perspective_projection_matrix">Perspective projection matrix</a> below for an introduction to how to use more complex matrices to help control and prevent clipping.</p>

<h3 id="Exercise">Exercise</h3>

<p>If that sounds a little abstract, open up the vertex shader and play around with the scale factor and watch how it shrinks vertices more towards the surface. Completely change the w component values for really trippy representations of space.</p>

<p>In the next section we'll take this step of copying Z into the w slot and turn it into a matrix.</p>

<h2 id="Simple_projection">Simple projection</h2>

<p>The last step of filling in the w component can actually be accomplished with a simple matrix. Start with the identity matrix:</p>

<pre class="brush: js notranslate">var identity = [
  1, 0, 0, 0,
  0, 1, 0, 0,
  0, 0, 1, 0,
  0, 0, 0, 1,
];

MDN.multiplyPoint(identity, [2, 3, 4, 1]);
//&gt; [2, 3, 4, 1]
</pre>

<p>Then move the last column's 1 up one space.</p>

<pre class="brush: js notranslate">var copyZ = [
  1, 0, 0, 0,
  0, 1, 0, 0,
  0, 0, 1, 1,
  0, 0, 0, 0,
];

MDN.multiplyPoint(copyZ, [2, 3, 4, 1]);
//&gt; [2, 3, 4, 4]
</pre>

<p>However in the last example we performed <code>(z + 1) * scaleFactor</code>:</p>

<pre class="notranslate">var scaleFactor = 0.5;

var simpleProjection = [
  1, 0, 0, 0,
  0, 1, 0, 0,
  0, 0, 1, scaleFactor,
  0, 0, 0, scaleFactor,
];

MDN.multiplyPoint(simpleProjection, [2, 3, 4, 1]);
//&gt; [2, 3, 4, 2.5]
</pre>

<p>Breaking it out a little further we can see how this works:</p>

<pre class="brush: js notranslate">var x = (2 * 1) + (3 * 0) + (4 * 0) + (1 * 0)
var y = (2 * 0) + (3 * 1) + (4 * 0) + (1 * 0)
var z = (2 * 0) + (3 * 0) + (4 * 1) + (1 * 0)
var w = (2 * 0) + (3 * 0) + (4 * scaleFactor) + (1 * scaleFactor)
</pre>

<p>The last line could be simplified to:</p>

<pre class="brush: js notranslate">w = (4 * scaleFactor) + (1 * scaleFactor)
</pre>

<p>Then factoring out the scaleFactor, we get this:</p>

<pre class="brush: js notranslate">w = (4 + 1) * scaleFactor
</pre>

<p>Which is exactly the same as the <code>(z + 1) * scaleFactor</code> that we used in the previous example.</p>

<p>In the box demo, an additional <code>computeSimpleProjectionMatrix()</code> method is added. This is called in the <code>draw()</code> method and has the scale factor passed to it. The result should be identical to the last example:</p>

<pre class="brush: js notranslate">CubeDemo.prototype.computeSimpleProjectionMatrix = function(scaleFactor) {
	this.transforms.projection = [
		1, 0, 0, 0,
		0, 1, 0, 0,
		0, 0, 1, scaleFactor,
		0, 0, 0, scaleFactor
	];
};
</pre>

<p>Although the result is identical, the important step here is in the vertex shader. Rather than modifying the vertex directly, it gets multiplied by an additional <strong><a href="#projection_matrix">projection matrix</a></strong>, which (as the name suggests) projects 3D points onto a 2D drawing surface:</p>

<pre class="brush: glsl notranslate">// Make sure to read the transformations in reverse order
gl_Position = projection * model * vec4(position, 1.0);
</pre>

<h3 id="The_results_4">The results</h3>

<p><a href="https://jsfiddle.net/zwyLLcbw">View on JSFiddle</a></p>

<p><img alt="A simple projection matrix" src="https://mdn.mozillademos.org/files/11381/part5.png" style="height: 531px; width: 800px;"></p>

<h2 id="The_viewing_frustum">The viewing frustum</h2>

<p>Before we move on to covering how to compute a perspective projection matrix, we need to introduce the concept of the <strong>{{interwiki("wikipedia", "viewing frustum")}}</strong> (also known as the <strong>view frustum</strong>). This is the region of space whose contents are visible to the user at the current time. It's the 3D region of space defined by the field of view and the distances specified as the nearest and farthest content that should be rendered.</p>

<p>While rendering, we need to determine which polygons need to be rendered in order to represent the scene. This is what the viewing frustum defines. But what's a frustum in the first place?</p>

<p>A {{interwiki("wikipedia", "frustum")}} is the 3D solid that results from taking any solid and slicing off two sections of it using two parallel planes. Consider our camera, which is viewing an area that starts immediately in front of its lens and extends off into the distance. The viewable area is a four-sided pyramid with its peak at the lens, its four sides corresponding to the extents of its peripheral vision range, and its base at the farthest distance it can see, like this:</p>

<div style="width: 42em; margin-bottom: 1em;"><img alt="A depiction of the entire viewing area of a camera. This area is a four-sided pyramid with its peak at the lens and its base at the world's maximum viewable distance." src="https://mdn.mozillademos.org/files/17295/FullCameraFOV.svg" style="display: block; margin: 0 auto; width: 36em;"></div>

<p>If we simply used this to determine the polygons to be rendered each frame, our renderer would need to render every polygon within this pyramid, all the way off into infinity, including also polygons that are very close to the lens—likely too close to be useful (and certainly including things that are so close that a real human wouldn't be able to focus on them in the same setting).</p>

<p>So the first step in reducing the number of polygons we need to compute and render, we turn this pyramid into the viewing frustum. The two planes we'll use to chop away vertices in order to reduce the polygon count are the <strong>near clipping plane</strong> and the <strong>far clipping plane</strong>.</p>

<p>In WebXR, the near and far clipping planes are defined by specifying the distance from the lens to the closest point on a plane which is perpendicular to the viewing direction. <span>Anything closer to the lens than the near clipping plane or farther from it than the far clipping plane is removed. This results in the viewing frustum, which looks like this:</span></p>

<div style="width: 42em; margin-bottom: 1em;"><img alt="A depiction of the camera's view frustum; the near and far planes have removed part of the volume, reducing the polygon count." src="https://mdn.mozillademos.org/files/17296/CameraViewFustum.svg" style="display: block; margin: 0 auto; width: 36em;"></div>

<p>The set of objects to be rendered for each frame is essentially created by starting with the set of all objects in the scene. Then any objects which are <em>entirely</em> outside the viewing frustum are removed from the set. Next, objects which partially extrude outside the viewing frustum are clipped by dropping any polygons which are entirely outside the frustum, and by clipping the polygons which cross outside the frustrum so that they no longer exit it.</p>

<p>Once that's been done, we have the largest set of polygons which are entirely within the viewing frustum. This list is usually further reduced using processes like {{interwiki("wikipedia", "back-face culling")}} (removing polygons whose back side is facing the camera) and occlusion culling using {{interwiki("wikipedia", "hidden-surface determination")}} (removing polygons which can't be seen because they're entirely blocked by polygons that are closer to the lens).</p>

<h2 id="Perspective_projection_matrix">Perspective projection matrix</h2>

<p>Up to this point, we've built up our own 3D rendering setup, step by step. However the current code as we've built it has some issues. For one, it gets skewed whenever we resize our window. Another is that our simple projection doesn't handle a wide range of values for the scene data. Most scenes don't work in clip space. It would be helpful to define what distance is relevant to the scene so that precision isn't lost in converting the numbers. Finally it's very helpful to have a fine-tuned control over what points get placed inside and outside of clip space. In the previous examples the corners of the cube occasionally get clipped.</p>

<p>The <strong>perspective projection matrix</strong> is a type of projection matrix that accomplishes all of these requirements. The math also starts to get a bit more involved and won't be fully explained in these examples. In short, it combines dividing by w (as done with the previous examples) with some ingenious manipulations based on <a href="https://en.wikipedia.org/wiki/Similarity_%28geometry%29">similar triangles</a>. If you want to read a full explanation of the math behind it check out some of the following links:</p>

<ul>
 <li><a href="http://www.songho.ca/opengl/gl_projectionmatrix.html">OpenGL Projection Matrix</a></li>
 <li><a href="http://ogldev.atspace.co.uk/www/tutorial12/tutorial12.html">Perspective Projection</a></li>
 <li><a href="http://stackoverflow.com/questions/28286057/trying-to-understand-the-math-behind-the-perspective-matrix-in-webgl/28301213#28301213">Trying to understand the math behind the perspective projection matrix in WebGL</a></li>
</ul>

<p>One important thing to note about the perspective projection matrix used below is that it flips the z axis. In clip space the z+ goes away from the viewer, while with this matrix it comes towards the viewer.</p>

<p>The reason to flip the z axis is that the clip space coordinate system is a left-handed coordinate system (wherein the z-axis points away from the viewer and into the screen), while the convention in mathematics, physics and 3D modeling, as well as for the view/eye coordinate system in OpenGL, is to use a right-handed coordinate system (z-axis points out of the screen towards the viewer) . More on that in the relevant Wikipedia articles: <a href="https://en.wikipedia.org/wiki/Cartesian_coordinate_system#Orientation_and_handedness">Cartesian coordinate system</a>, <a href="https://en.wikipedia.org/wiki/Right-hand_rule">Right-hand rule</a>.</p>

<p>Let's take a look at a <code>perspectiveMatrix()</code> function, which computes the perspective projection matrix.</p>

<pre class="brush:js notranslate">MDN.perspectiveMatrix = function(fieldOfViewInRadians, aspectRatio, near, far) {
  var f = 1.0 / Math.tan(fieldOfViewInRadians / 2);
  var rangeInv = 1 / (near - far);

  return [
    f / aspectRatio, 0,                          0,   0,
    0,               f,                          0,   0,
    0,               0,    (near + far) * rangeInv,  -1,
    0,               0,  near * far * rangeInv * 2,   0
  ];
}
</pre>

<p>The four parameters into this function are:</p>

<dl>
 <dt><code>fieldOfviewInRadians</code></dt>
 <dd>An angle, given in radians, indicating how much of the scene is visible to the viewer at once. The larger the number is, the more is visible by the camera. The geometry at the edges becomes more and more distorted, equivalent to a wide angle lens. When the field of view is larger, the objects typically get smaller. When the field of view is smaller, then the camera can see less and less in the scene. The objects are distorted much less by perspective and objects seem much closer to the camera</dd>
 <dt><code>aspectRatio</code></dt>
 <dd>The scene's aspect ratio, which is equivalent to its width divided by its height. In these examples, that's the window's width divided by the window height. The introduction of this parameter finally solves the problem wherein the model gets warped as the canvas is resized and reshaped.</dd>
 <dt><code>nearClippingPlaneDistance</code></dt>
 <dd>A positive number indicating the distance into the screen to a plane which is perpendicular to the floor, nearer than which everything gets clipped away. This is mapped to -1 in clip space, and should not be set to 0.</dd>
 <dt><code>farClippingPlaneDistance</code></dt>
 <dd>A positive number indicating the distance to the plane beyond which geometry is clipped away. This is mapped to 1 in clip space. This value should be kept reasonably close to the distance of the geometry in order to avoid precision errors creeping in while rendering.</dd>
</dl>

<p>In the latest version of the box demo, the <code>computeSimpleProjectionMatrix()</code> method has been replaced with the <code>computePerspectiveMatrix()</code> method.</p>

<pre class="brush: js notranslate">CubeDemo.prototype.computePerspectiveMatrix = function() {
  var fieldOfViewInRadians = Math.PI * 0.5;
  var aspectRatio = window.innerWidth / window.innerHeight;
  var nearClippingPlaneDistance = 1;
  var farClippingPlaneDistance = 50;

  this.transforms.projection = MDN.perspectiveMatrix(
    fieldOfViewInRadians,
    aspectRatio,
    nearClippingPlaneDistance,
    farClippingPlaneDistance
  );
};
</pre>

<p>The shader code is identical to the previous example:</p>

<pre class="brush: js notranslate">gl_Position = projection * model * vec4(position, 1.0);
</pre>

<p>Additionally (not shown), the position and scale matrices of the model have been changed to take it out of clip space and into the larger coordinate system.</p>

<h3 id="The_results_5">The results</h3>

<p><a href="https://jsfiddle.net/Lzxw7e1q">View on JSFiddle</a></p>

<p><img alt="A true perspective matrix" src="https://mdn.mozillademos.org/files/11383/part6.png" style="height: 531px; width: 800px;"></p>

<h3 id="Exercises_3">Exercises</h3>

<ul>
 <li>Experiment with the parameters of the perspective projection matrix and the model matrix.</li>
 <li>Swap out the perspective projection matrix to use {{interwiki("wikipedia", "orthographic projection")}}. In the MDN WebGL shared code you'll find the <code>MDN.orthographicMatrix()</code>. This can replace the <code>MDN.perspectiveMatrix()</code> function in <code>CubeDemo.prototype.computePerspectiveMatrix()</code>.</li>
</ul>

<h2 id="View_matrix">View matrix</h2>

<p>While some graphics libraries have a virtual camera that can be positioned and pointed while composing a scene, OpenGL (and by extension WebGL) does not. This is where the <strong>view matrix</strong> comes in. Its job is to translate, rotate, and scale the objects in the scene so that they are located in the right place relative to the viewer given the viewer's position and orientation.</p>

<h3 id="Simulating_a_camera">Simulating a camera</h3>

<p>This makes use of one of the fundamental facets of Einstein's special relativity theory: the principle of reference frames and relative motion says that, from the perspective of a viewer, you can simulate changing the position and orientation of the viewer by applying the opposite change to the objects in the scene. Either way, the result appears to be identical to the viewer.</p>

<p>Consider a box sitting on a table and a camera resting on the table one meter away, pointed at the box, the front of which is pointed toward the camera. Then consider moving the camera away from the box until it's two meters away (by adding a meter to the camera's Z position), then sliding it 10 centimeters to the its left. The box recedes from the camera by that amount and slides to the right slightly, thereby appearing smaller to the camera and exposing a small amount of its left side to the camera.</p>

<p>Now let's reset the scene, placing the box back in its starting point, with the camera two meters from, and directly facing, the box. This time, however, the camera is locked down on the table and cannot be moved or turned. This is what working in WebGL is like. So how do we simulate moving the camera through space?</p>

<p>Instead of moving the camera backward and to the left, we apply the inverse transform to the box: we move the <em>box</em> backward one meter, and then 10 centimeters to its right. The result, from the perspective of each of the two objects, is identical.</p>

<p><strong>&lt;&lt;&lt; insert image(s) here &gt;&gt;&gt;</strong></p>

<p>The final step in all of this is to create the <strong>view matrix</strong>, which transforms the objects in the scene so they're positioned to simulate the camera's current location and orientation. Our code as it stands can move the cube around in world space and project everything to have perspective, but we still can't move the camera.</p>

<p>Imagine shooting a movie with a physical camera. You have the freedom to place the camera essentially anywhere you wish, and to aim the camera in whichever direction you choose. To simulate this in 3D graphics, we use a view matrix to simulate the position and rotation of that physical camera.</p>

<p>Unlike the model matrix, which directly transforms the model vertices, the view matrix moves an abstract camera around. In reality, the vertex shader is still only moving the models while the "camera" stays in place. In order for this to work out correctly, the inverse of the transform matrix must be used. The inverse matrix essentially reverses a transformation, so if we move the camera view forward, the inverse matrix causes the objects in the scene to move back.</p>

<p>The following <code>computeViewMatrix()</code> method animates the view matrix by moving it in and out, and left and right.</p>

<pre class="brush: js notranslate">CubeDemo.prototype.computeViewMatrix = function(now) {
  var moveInAndOut = 20 * Math.sin(now * 0.002);
  var moveLeftAndRight = 15 * Math.sin(now * 0.0017);

  // Move the camera around
  var position = MDN.translateMatrix(moveLeftAndRight, 0, 50 + moveInAndOut );

  // Multiply together, make sure and read them in opposite order
  var matrix = MDN.multiplyArrayOfMatrices([
    // Exercise: rotate the camera view
    position
  ]);

  // Inverse the operation for camera movements, because we are actually
  // moving the geometry in the scene, not the camera itself.
  this.transforms.view = MDN.invertMatrix(matrix);
};
</pre>

<p>The shader now uses three matrices.</p>

<pre class="brush: glsl notranslate">gl_Position = projection * view * model * vec4(position, 1.0);
</pre>

<p>After this step, the GPU pipeline will clip the out of range vertices, and send the model down to the fragment shader for rasterization.</p>

<h3 id="The_results_6">The results</h3>

<p><a href="https://jsfiddle.net/86fd797g">View on JSFiddle</a></p>

<p><img alt="The view matrix" src="https://mdn.mozillademos.org/files/11385/part7.png" style="height: 531px; width: 800px;"></p>

<h3 id="Relating_the_coordinate_systems">Relating the coordinate systems</h3>

<p>At this point it would be beneficial to take a step back and look at and label the various coordinate systems we use. First off, the cube's vertices are defined in <strong>model space</strong>. To move the model around the scene. these vertices need to be converted into <strong>world space</strong> by applying the model matrix.</p>

<p>model space → model matrix → world space</p>

<p>The camera hasn't done anything yet, and the points need to be moved again. Currently they are in world space, but they need to be moved to <strong>view space</strong> (using the view matrix) in order to represent the camera placement.</p>

<p>world space → view matrix → view space</p>

<p>Finally a <strong>projection</strong> (in our case the perspective projection matrix) needs to be added in order to map the world coordinates into clip space coordinates.</p>

<p>view space → projection matrix → clip space</p>

<h3 id="Exercise_2">Exercise</h3>

<ul>
 <li>Move the camera around the scene.</li>
 <li>Add some rotation matrices to the view matrix to look around.</li>
 <li>Finally, track the mouse's position. Use 2 rotation matrices to have the camera look up and down based on where the user's mouse is on the screen.</li>
</ul>

<h2 id="See_also">See also</h2>

<ul>
 <li><a href="/ja/docs/Web/API/WebGL_API">WebGL</a></li>
 <li>{{interwiki("wikipedia", "3D projection")}}</li>
</ul>