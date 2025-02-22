```@meta
EditURL = "/tutorials/AutomaticDifferentiation.jl"
```
```@raw html
<style>
    table {
        display: table !important;
        margin: 2rem auto !important;
        border-top: 2pt solid rgba(0,0,0,0.2);
        border-bottom: 2pt solid rgba(0,0,0,0.2);
    }

    pre, div {
        margin-top: 1.4rem !important;
        margin-bottom: 1.4rem !important;
    }

    .code-output {
        padding: 0.7rem 0.5rem !important;
    }

    .admonition-body {
        padding: 0em 1.25em !important;
    }
</style>

<!-- PlutoStaticHTML.Begin -->
<!--
    # This information is used for caching.
    [PlutoStaticHTML.State]
    input_sha = "47c78bffb5acbfe4710f85019af91f48c82c7ac040c2df1cf5b453592f73e2d6"
    julia_version = "1.8.4"
-->

<div class="markdown"><h1>Using (Euclidean) AD in Manopt.jl</h1></div>


<div class="markdown"><p>Since <a href="https://juliamanifolds.github.io/Manifolds.jl/latest/">Manifolds.jl</a> 0.7, the support of automatic differentiation support has been extended.</p><p>This tutorial explains how to use Euclidean tools to derive a gradient for a real-valued function <span class="tex">$f\colon \mathcal M → ℝ$</span>. We will consider two methods: an intrinsic variant and a variant employing the embedding. These gradients can then be used within any gradient based optimization algorithm in <a href="https://manoptjl.org">Manopt.jl</a>.</p><p>While by default we use <a href="https://juliadiff.org/FiniteDifferences.jl/latest/">FiniteDifferences.jl</a>, you can also use <a href="https://github.com/JuliaDiff/FiniteDiff.jl">FiniteDiff.jl</a>, <a href="https://juliadiff.org/ForwardDiff.jl/stable/">ForwardDiff.jl</a>, <a href="https://juliadiff.org/ReverseDiff.jl/">ReverseDiff.jl</a>, or  <a href="https://fluxml.ai/Zygote.jl/">Zygote.jl</a>.</p></div>


<div class="markdown"><p>In this Notebook we will take a look at a few possibilities to approximate or derive the gradient of a function <span class="tex">$f:\mathcal M \to ℝ$</span> on a Riemannian manifold, without computing it yourself. There are mainly two different philosophies:</p><ol><li><p>Working <em>instrinsically</em>, i.e. staying on the manifold and in the tangent spaces. Here, we will consider approximating the gradient by forward differences.</p></li><li><p>Working in an embedding – there we can use all tools from functions on Euclidean spaces – finite differences or automatic differenciation – and then compute the corresponding Riemannian gradient from there.</p></li></ol></div>


```
## Setup
```@raw html
<div class="markdown">
<p>If you open this notebook in Pluto locally it switches between two modes. If the tutorial is within the <code>Manopt.jl</code> repository, this notebook tries to use the local package in development mode. Otherwise, the file uses the Pluto pacakge management version. In this case, the includsion of images might be broken. unless you create a subfolder <code>optimize</code> and activate <code>asy</code>-rendering.</p></div>








<div class="markdown"><p>Since the loading is a little complicated, we show, which versions of packages were installed in the following.</p></div>

<pre class='language-julia'><code class='language-julia'>with_terminal() do
    Pkg.status()
end</code></pre>
<pre id="plutouiterminal">�[32m�[1mStatus�[22m�[39m `/private/var/folders/_v/wg192lpd3mb1lp55zz7drpcw0000gn/T/jl_hpuuJD/Project.toml`
 �[90m [26cc04aa] �[39mFiniteDifferences v0.12.26
 �[90m [af67fdf4] �[39mManifoldDiff v0.2.1
 �[90m [1cead3c2] �[39mManifolds v0.8.44
 �[90m [0fc0a36d] �[39mManopt v0.4.0 `~/Repositories/Julia/Manopt.jl`
 �[90m [7f904dfe] �[39mPlutoUI v0.7.49
 �[90m [37e2e46d] �[39mLinearAlgebra
 �[90m [44cfe95a] �[39mPkg v1.8.0
 �[90m [9a3f8284] �[39mRandom
</pre>


```
## 1. (Intrinsic) Forward Differences
```@raw html
<div class="markdown">
<p>A first idea is to generalize (multivariate) finite differences to Riemannian manifolds. Let <span class="tex">$X_1,\ldots,X_d ∈ T_p\mathcal M$</span> denote an orthonormal basis of the tangent space <span class="tex">$T_p\mathcal M$</span> at the point <span class="tex">$p∈\mathcal M$</span> on the Riemannian manifold.</p><p>We can generalize the notion of a directional derivative, i.e. for the “direction” <span class="tex">$Y∈T_p\mathcal M$</span>. Let <span class="tex">$c\colon [-ε,ε]$</span>, <span class="tex">$ε&gt;0$</span>, be a curve with <span class="tex">$c(0) = p$</span>, <span class="tex">$\dot c(0) = Y$</span>, e.g. <span class="tex">$c(t)= \exp_p(tY)$</span>. We obtain</p><p class="tex">$$	Df(p)[Y] = \left. \frac{d}{dt} \right|_{t=0} f(c(t)) = \lim_{t \to 0} \frac{1}{t}(f(\exp_p(tY))-f(p))$$</p><p>We can approximate <span class="tex">$Df(p)[X]$</span> by a finite difference scheme for an <span class="tex">$h&gt;0$</span> as</p><p class="tex">$$DF(p)[Y] ≈ G_h(Y) := \frac{1}{h}(f(\exp_p(hY))-f(p))$$</p><p>Furthermore the gradient <span class="tex">$\operatorname{grad}f$</span> is the Riesz representer of the differential, ie.</p><p class="tex">$$	Df(p)[Y] = g_p(\operatorname{grad}f(p), Y),\qquad \text{ for all } Y ∈ T_p\mathcal M$$</p><p>and since it is a tangent vector, we can write it in terms of a basis as</p><p class="tex">$$	\operatorname{grad}f(p) = \sum_{i=1}^{d} g_p(\operatorname{grad}f(p),X_i)X_i
	= \sum_{i=1}^{d} Df(p)[X_i]X_i$$</p><p>and perform the approximation from above to obtain</p><p class="tex">$$	\operatorname{grad}f(p) ≈ \sum_{i=1}^{d} G_h(X_i)X_i$$</p><p>for some suitable step size <span class="tex">$h$</span>. This comes at the cost of <span class="tex">$d+1$</span> function evaluations and <span class="tex">$d$</span> exponential maps.</p></div>


<div class="markdown"><p>This is the first variant we can use. An advantage is that it is <em>intrinsic</em> in the sense that it does not require any embedding of the manifold.</p></div>


<div class="markdown"><h3>An Example: The Rayleigh Quotient</h3><p>The Rayleigh quotient is concerned with finding eigenvalues (and eigenvectors) of a symmetric matrix <span class="tex">$A\in ℝ^{(n+1)×(n+1)}$</span>. The optimization problem reads</p><p class="tex">$$F\colon ℝ^{n+1} \to ℝ,\quad F(\mathbf x) = \frac{\mathbf x^\mathrm{T}A\mathbf x}{\mathbf x^\mathrm{T}\mathbf x}$$</p><p>Minimizing this function yields the smallest eigenvalue <span class="tex">$\lambda_1$</span> as a value and the corresponding minimizer <span class="tex">$\mathbf x^*$</span> is a corresponding eigenvector.</p><p>Since the length of an eigenvector is irrelevant, there is an ambiguity in the cost function. It can be better phrased on the sphere <span class="tex">$𝕊^n$</span> of unit vectors in <span class="tex">$\mathbb R^{n+1}$</span>, i.e.</p><p class="tex">$$\operatorname*{arg\,min}_{p \in 𝕊^n}\ f(p) = \operatorname*{arg\,min}_{\ p \in 𝕊^n} p^\mathrm{T}Ap$$</p><p>We can compute the Riemannian gradient exactly as</p><p class="tex">$$\operatorname{grad} f(p) = 2(Ap - pp^\mathrm{T}Ap)$$</p><p>so we can compare it to the approximation by finite differences.</p></div>

<pre class='language-julia'><code class='language-julia'>begin
    Random.seed!(42)
    n = 200
    A = randn(n + 1, n + 1)
    A = Symmetric(A)
    M = Sphere(n)
    nothing
end</code></pre>


<pre class='language-julia'><code class='language-julia'>f1(p) = p' * A'p</code></pre>
<pre class="code-output documenter-example-output" id="var-f1">f1 (generic function with 1 method)</pre>

<pre class='language-julia'><code class='language-julia'>gradf1(p) = 2 * (A * p - p * p' * A * p)</code></pre>
<pre class="code-output documenter-example-output" id="var-gradf1">gradf1 (generic function with 1 method)</pre>


<div class="markdown"><p>Manifolds provides a finite difference scheme in tangent spaces, that you can introduce to use an existing framework (if the wrapper is implemented) form Euclidean space. Here we use <code>FiniteDiff.jl</code>.</p></div>

<pre class='language-julia'><code class='language-julia'>r_backend = ManifoldDiff.TangentDiffBackend(ManifoldDiff.FiniteDifferencesBackend())</code></pre>
<pre class="code-output documenter-example-output" id="var-r_backend">ManifoldDiff.TangentDiffBackend{ManifoldDiff.FiniteDifferencesBackend{FiniteDifferences.AdaptedFiniteDifferenceMethod{5, 1, FiniteDifferences.UnadaptedFiniteDifferenceMethod{7, 5}}}, ExponentialRetraction, LogarithmicInverseRetraction, DefaultOrthonormalBasis{ℝ, ManifoldsBase.TangentSpaceType}, DefaultOrthonormalBasis{ℝ, ManifoldsBase.TangentSpaceType}}(ManifoldDiff.FiniteDifferencesBackend{FiniteDifferences.AdaptedFiniteDifferenceMethod{5, 1, FiniteDifferences.UnadaptedFiniteDifferenceMethod{7, 5}}}(FiniteDifferences.AdaptedFiniteDifferenceMethod{5, 1, FiniteDifferences.UnadaptedFiniteDifferenceMethod{7, 5}}([-2, -1, 0, 1, 2], [0.08333333333333333, -0.6666666666666666, 0.0, 0.6666666666666666, -0.08333333333333333], ([-0.08333333333333333, 0.5, -1.5, 0.8333333333333334, 0.25], [0.08333333333333333, -0.6666666666666666, 0.0, 0.6666666666666666, -0.08333333333333333], [-0.25, -0.8333333333333334, 1.5, -0.5, 0.08333333333333333]), 10.0, 1.0, Inf, 0.05555555555555555, 1.4999999999999998, FiniteDifferences.UnadaptedFiniteDifferenceMethod{7, 5}([-3, -2, -1, 0, 1, 2, 3], [-0.5, 2.0, -2.5, 0.0, 2.5, -2.0, 0.5], ([0.5, -4.0, 12.5, -20.0, 17.5, -8.0, 1.5], [-0.5, 2.0, -2.5, 0.0, 2.5, -2.0, 0.5], [-1.5, 8.0, -17.5, 20.0, -12.5, 4.0, -0.5]), 10.0, 1.0, Inf, 0.5365079365079365, 10.0))), ExponentialRetraction(), LogarithmicInverseRetraction(), DefaultOrthonormalBasis(ℝ), DefaultOrthonormalBasis(ℝ))</pre>

<pre class='language-julia'><code class='language-julia'>gradf1_FD(p) = ManifoldDiff.gradient(M, f1, p, r_backend)</code></pre>
<pre class="code-output documenter-example-output" id="var-gradf1_FD">gradf1_FD (generic function with 1 method)</pre>

<pre class='language-julia'><code class='language-julia'>begin
    p = zeros(n + 1)
    p[1] = 1.0
    X1 = gradf1(p)
    X2 = gradf1_FD(p)
    norm(M, p, X1 - X2)
end</code></pre>
<pre class="code-output documenter-example-output" id="var-p">1.0183547605045089e-12</pre>


<div class="markdown"><p>We obtain quite a good approximation of the gradient.</p></div>


```
## 2. Conversion of a Euclidean Gradient in the Embedding to a Riemannian Gradient of a (not Necessarily Isometrically) Embedded Manifold
```@raw html
<div class="markdown">
<p>Let <span class="tex">$\tilde f\colon\mathbb R^m \to \mathbb R$</span> be a function on the embedding of an <span class="tex">$n$</span>-dimensional manifold <span class="tex">$\mathcal M \subset \mathbb R^m$</span> and let <span class="tex">$f\colon \mathcal M \to \mathbb R$</span> denote the restriction of <span class="tex">$\tilde f$</span> to the manifold <span class="tex">$\mathcal M$</span>.</p><p>Since we can use the pushforward of the embedding to also embed the tangent space <span class="tex">$T_p\mathcal M$</span>, <span class="tex">$p\in \mathcal M$</span>, we can similarly obtain the differential <span class="tex">$Df(p)\colon T_p\mathcal M \to \mathbb R$</span> by restricting the differential <span class="tex">$D\tilde f(p)$</span> to the tangent space.</p><p>If both <span class="tex">$T_p\mathcal M$</span> and <span class="tex">$T_p\mathbb R^m$</span> have the same inner product, or in other words the manifold is isometrically embedded in <span class="tex">$\mathbb R^m$</span> (like for example the sphere <span class="tex">$\mathbb S^n\subset\mathbb R^{m+1}$</span>), then this restriction of the differential directly translates to a projection of the gradient, i.e.</p><p class="tex">$$\operatorname{grad}f(p) = \operatorname{Proj}_{T_p\mathcal M}(\operatorname{grad} \tilde f(p))$$</p><p>More generally we might have to take a change of the metric into account, i.e.</p><p class="tex">$$\langle  \operatorname{Proj}_{T_p\mathcal M}(\operatorname{grad} \tilde f(p)), X \rangle
= Df(p)[X] = g_p(\operatorname{grad}f(p), X)$$</p><p>or in words: we have to change the Riesz representer of the (restricted/projected) differential of <span class="tex">$f$</span> (<span class="tex">$\tilde f$</span>) to the one with respect to the Riemannian metric. This is done using <a href="https://juliamanifolds.github.io/Manifolds.jl/latest/manifolds/metric.html#Manifolds.change_representer-Tuple{AbstractManifold,%20AbstractMetric,%20Any,%20Any}"><code>change_representer</code></a>.</p></div>


<div class="markdown"><h3>A Continued Example</h3><p>We continue with the Rayleigh Quotient from before, now just starting with the defintion of the Euclidean case in the embedding, the function <span class="tex">$F$</span>.</p></div>

<pre class='language-julia'><code class='language-julia'>F(x) = x' * A * x / (x' * x);</code></pre>



<div class="markdown"><p>The cost function is the same by restriction</p></div>

<pre class='language-julia'><code class='language-julia'>f2(M, p) = F(p);</code></pre>



<div class="markdown"><p>The gradient is now computed combining our gradient scheme with FiniteDifferences.</p></div>

<pre class='language-julia'><code class='language-julia'>function grad_f2_AD(M, p)
    return Manifolds.gradient(
        M, F, p, Manifolds.RiemannianProjectionBackend(ManifoldDiff.FiniteDifferencesBackend())
    )
end</code></pre>
<pre class="code-output documenter-example-output" id="var-grad_f2_AD">grad_f2_AD (generic function with 1 method)</pre>

<pre class='language-julia'><code class='language-julia'>X3 = grad_f2_AD(M, p)</code></pre>
<pre class="code-output documenter-example-output" id="var-X3">201-element Vector{Float64}:
  0.0
  0.14038510230588766
  1.2081562751116681
 -1.8650092022183193
 -1.3296034534632897
  0.8233596435551424
  0.013307949254910802
  ⋮
 -1.0895103349140873
  2.423453243905215
  2.349106449107357
  0.5799736335284804
 -0.07423232665910076
  2.0859147085739465</pre>

<pre class='language-julia'><code class='language-julia'>norm(M, p, X1 - X3)</code></pre>
<pre class="code-output documenter-example-output" id="var-hash230331">1.7211299825309384e-12</pre>


<div class="markdown"><h3>An Example for a Nonisometrically Embedded Manifold</h3><p>on the manifold <span class="tex">$\mathcal P(3)$</span> of symmetric positive definite matrices.</p></div>


<div class="markdown"><p>The following function computes (half) the distance squared (with respect to the linear affine metric) on the manifold <span class="tex">$\mathcal P(3)$</span> to the identity, i.e. <span class="tex">$I_3$</span>. Denoting the unit matrix we consider the function</p><p class="tex">$$	G(q) = \frac{1}{2}d^2_{\mathcal P(3)}(q,I_3) = \lVert \operatorname{Log}(q) \rVert_F^2,$$</p><p>where <span class="tex">$\operatorname{Log}$</span> denotes the matrix logarithm and <span class="tex">$\lVert \cdot \rVert_F$</span> is the Frobenius norm. This can be computed for symmetric positive definite matrices by summing the squares of the logarithms of the eigenvalues of <span class="tex">$q$</span> and dividing by two:</p></div>

<pre class='language-julia'><code class='language-julia'>G(q) = sum(log.(eigvals(Symmetric(q))) .^ 2) / 2</code></pre>
<pre class="code-output documenter-example-output" id="var-G">G (generic function with 1 method)</pre>


<div class="markdown"><p>We can also interpret this as a function on the space of matrices and apply the Euclidean finite differences machinery; in this way we can easily derive the Euclidean gradient. But when computing the Riemannian gradient, we have to change the representer (see again <a href="https://juliamanifolds.github.io/Manifolds.jl/latest/manifolds/metric.html#Manifolds.change_representer-Tuple{AbstractManifold,%20AbstractMetric,%20Any,%20Any}"><code>change_representer</code></a>) after projecting onto the tangent space <span class="tex">$T_p\mathcal P(n)$</span> at <span class="tex">$p$</span>.</p><p>Let's first define a point and the manifold <span class="tex">$N=\mathcal P(3)$</span>.</p></div>

<pre class='language-julia'><code class='language-julia'>rotM(α) = [1.0 0.0 0.0; 0.0 cos(α) sin(α); 0.0 -sin(α) cos(α)]</code></pre>
<pre class="code-output documenter-example-output" id="var-rotM">rotM (generic function with 1 method)</pre>

<pre class='language-julia'><code class='language-julia'>q = rotM(π / 6) * [1.0 0.0 0.0; 0.0 2.0 0.0; 0.0 0.0 3.0] * transpose(rotM(π / 6))</code></pre>
<pre class="code-output documenter-example-output" id="var-q">3×3 Matrix{Float64}:
 1.0  0.0       0.0
 0.0  2.25      0.433013
 0.0  0.433013  2.75</pre>

<pre class='language-julia'><code class='language-julia'>N = SymmetricPositiveDefinite(3)</code></pre>
<pre class="code-output documenter-example-output" id="var-N">SymmetricPositiveDefinite(3)</pre>

<pre class='language-julia'><code class='language-julia'>is_point(N, q)</code></pre>
<pre class="code-output documenter-example-output" id="var-hash141872">true</pre>


<div class="markdown"><p>We could first just compute the gradient using <code>FiniteDifferences.jl</code>, but this yields the Euclidean gradient:</p></div>

<pre class='language-julia'><code class='language-julia'>FiniteDifferences.grad(central_fdm(5, 1), G, q)</code></pre>
<pre class="code-output documenter-example-output" id="var-hash184002">([3.240417492806275e-14 1.1767558621318606e-14 2.075425785233536e-14; 0.0 0.35148121676557886 0.017000516835447246; 0.0 0.0 0.3612964697372195],)</pre>


<div class="markdown"><p>Instead, we use the <a href="https://juliamanifolds.github.io/Manifolds.jl/latest/features/differentiation.html#Manifolds.RiemannianProjectionBackend"><code>RiemannianProjectedBackend</code></a> of <code>Manifolds.jl</code>, which in this case internally uses <code>FiniteDifferences.jl</code> to compute a Euclidean gradient but then uses the conversion explained above to derive the Riemannian gradient.</p><p>We define this here again as a function <code>grad_G_FD</code> that could be used in the <code>Manopt.jl</code> framework within a gradient based optimization.</p></div>

<pre class='language-julia'><code class='language-julia'>function grad_G_FD(N, q)
    return Manifolds.gradient(
        N, G, q, ManifoldDiff.RiemannianProjectionBackend(ManifoldDiff.FiniteDifferencesBackend())
    )
end</code></pre>
<pre class="code-output documenter-example-output" id="var-grad_G_FD">grad_G_FD (generic function with 1 method)</pre>

<pre class='language-julia'><code class='language-julia'>G1 = grad_G_FD(N, q)</code></pre>
<pre class="code-output documenter-example-output" id="var-G1">3×3 Matrix{Float64}:
 3.24042e-14  1.77319e-14  3.10849e-14
 1.77319e-14  1.86368      0.826856
 3.10849e-14  0.826856     2.81845</pre>


<div class="markdown"><p>Now, we can again compare this to the (known) solution of the gradient, namely the gradient of (half of) the distance squared, i.e. <span class="tex">$G(q) = \frac{1}{2}d^2_{\mathcal P(3)}(q,I_3)$</span> is given by <span class="tex">$\operatorname{grad} G(q) = -\operatorname{log}_q I_3$</span>, where <span class="tex">$\operatorname{log}$</span> is the <a href="https://juliamanifolds.github.io/Manifolds.jl/latest/manifolds/symmetricpositivedefinite.html#Base.log-Tuple{SymmetricPositiveDefinite,%20Vararg{Any,%20N}%20where%20N}">logarithmic map</a> on the manifold.</p></div>

<pre class='language-julia'><code class='language-julia'>G2 = -log(N, q, Matrix{Float64}(I, 3, 3))</code></pre>
<pre class="code-output documenter-example-output" id="var-G2">3×3 Matrix{Float64}:
 -0.0  -0.0       -0.0
 -0.0   1.86368    0.826856
 -0.0   0.826856   2.81845</pre>


<div class="markdown"><p>Both terms agree up to <span class="tex">$1.8×10^{-12}$</span>:</p></div>

<pre class='language-julia'><code class='language-julia'>norm(G1 - G2)</code></pre>
<pre class="code-output documenter-example-output" id="var-hash135981">1.4601907912994879e-12</pre>

<pre class='language-julia'><code class='language-julia'>isapprox(M, q, G1, G2; atol=2 * 1e-12)</code></pre>
<pre class="code-output documenter-example-output" id="var-hash251789">true</pre>


```
## Summary
```@raw html
<div class="markdown">
<p>This tutorial illustrates how to use tools from Euclidean spaces, finite differences or automatic differentiation, to compute gradients on Riemannian manifolds. The scheme allows to use <em>any</em> differentiation framework within the embedding to derive a Riemannian gradient.</p></div>

<!-- PlutoStaticHTML.End -->
```
