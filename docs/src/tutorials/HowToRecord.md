```@meta
EditURL = "/tutorials/HowToRecord.jl"
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
    input_sha = "48d5aa646badbf3926bd3507783086036b8c99ee6386c7176fc8b8625b7f057d"
    julia_version = "1.8.4"
-->

<div class="markdown"><h1>How to Record Data During the Iterations</h1><p>The recording and debugging features make it possible to record nearly any data during the iterations. This tutorial illustrates how to:</p><ul><li><p>record one value during the iterations;</p></li><li><p>record multiple values during the iterations and access them afterwards;</p></li><li><p>define an own <code>RecordAction</code> to perform individual recordings.</p></li></ul><p>Several predefined recordings exist, for example <code>RecordCost()</code> or <code>RecordGradient()</code>, depending on the solver used. For fields of the <code>State</code> this can be directly done using the [<code>RecordEntry(:field)</code>]. For other recordings, for example more advanced computations before storing a value, an own <code>RecordAction</code> can be defined.</p><p>We illustrate these using the gradient descent from the mean computation tutorial.</p><p>This tutorial is a <a href="https://github.com/fonsp/Pluto.jl">Pluto 🎈 notebook</a>, , so if you are reading the <code>Manopt.jl</code> documentation you can also <a href="https://github.com/JuliaManifolds/Manopt.jl/raw/master/tutorials/HowToRecord.jl">download</a> the notebook and run it yourself within Pluto.</p></div>


```
## Setup
```@raw html
<div class="markdown">
<p>If you open this notebook in Pluto locally it switches between two modes. If the tutorial is within the <code>Manopt.jl</code> repository, this notebook tries to use the local package in development mode. Otherwise, the file uses the Pluto pacakge management version.</p></div>








<div class="markdown"><p>Now we can set up our small test example, which is just a gradient descent for the Riemannian center of mass, see the <a href="https://manoptjl.org/stable/tutorials/Optimize!/">Get Started: Optimize!</a> tutorial for details.</p><p>Here we focus on ways to investigate the behaviour during iterations by using Recording techniques.</p></div>


<div class="markdown"><p>Since the loading is a little complicated, we show, which versions of packages were installed in the following.</p></div>

<pre class='language-julia'><code class='language-julia'>with_terminal() do
    Pkg.status()
end</code></pre>
<pre id="plutouiterminal">�[32m�[1mStatus�[22m�[39m `/private/var/folders/_v/wg192lpd3mb1lp55zz7drpcw0000gn/T/jl_XHZgxw/Project.toml`
 �[90m [1cead3c2] �[39mManifolds v0.8.42
 �[90m [0fc0a36d] �[39mManopt v0.4.0 `~/Repositories/Julia/Manopt.jl`
 �[90m [7f904dfe] �[39mPlutoUI v0.7.49
 �[90m [44cfe95a] �[39mPkg v1.8.0
 �[90m [9a3f8284] �[39mRandom
</pre>

<pre class='language-julia'><code class='language-julia'>begin
    Random.seed!(42)
    m = 30
    M = Sphere(m)
    n = 800
    σ = π / 8
    x = zeros(Float64, m + 1)
    x[2] = 1.0
    data = [exp(M, x, σ * rand(M; vector_at=x)) for i in 1:n]
end</code></pre>
<pre class="code-output documenter-example-output" id="var-m">800-element Vector{Vector{Float64}}:
 [-0.054658825167894595, -0.5592077846510423, -0.04738273828111257, -0.04682080720921302, 0.12279468849667038, 0.07171438895366239, -0.12930045409417057, -0.22102081626380404, -0.31805333254577767, 0.0065859500152017645  …  -0.21999168261518043, 0.19570142227077295, 0.340909965798364, -0.0310802190082894, -0.04674431076254687, -0.006088297671169996, 0.01576037011323387, -0.14523596850249543, 0.14526158060820338, 0.1972125856685378]
 [-0.08192376929745249, -0.5097715132187676, -0.008339904915541005, 0.07289741328038676, 0.11422036270613797, -0.11546739299835748, 0.2296996932628472, 0.1490467170835958, -0.11124820565850364, -0.11790721606521781  …  -0.16421249630470344, -0.2450575844467715, -0.07570080850379841, -0.07426218324072491, -0.026520181327346338, 0.11555341205250205, -0.0292955762365121, -0.09012096853677576, -0.23470556634911574, -0.026214242996704013]
 [-0.22951484264859257, -0.6083825348640186, 0.14273766477054015, -0.11947823367023377, 0.05984293499234536, 0.058820835498203126, 0.07577331705863266, 0.1632847202946857, 0.20244385489915745, 0.04389826920203656  …  0.3222365119325929, 0.009728730325524067, -0.12094785371632395, -0.36322323926212824, -0.0689253407939657, 0.23356953371702974, 0.23489531397909744, 0.078303336494718, -0.14272984135578806, 0.07844539956202407]
 [-0.0012588500237817606, -0.29958740415089763, 0.036738459489123514, 0.20567651907595125, -0.1131046432541904, -0.06032435985370224, 0.3366633723165895, -0.1694687746143405, -0.001987171245125281, 0.04933779858684409  …  -0.2399584473006256, 0.19889267065775063, 0.22468755918787048, 0.1780090580180643, 0.023703860700539356, -0.10212737517121755, 0.03807004103115319, -0.20569120952458983, -0.03257704254233959, 0.06925473452536687]
 [-0.035534309946938375, -0.06645560787329002, 0.14823972268208874, -0.23913346587232426, 0.038347027875883496, 0.10453333143286662, 0.050933995140290705, -0.12319549375687473, 0.12956684644537844, -0.23540367869989412  …  -0.41471772859912864, -0.1418984610380257, 0.0038321446836859334, 0.23655566917750157, -0.17500681300994742, -0.039189751036839374, -0.08687860620942896, -0.11509948162959047, 0.11378233994840942, 0.38739450723013735]
 [-0.3122539912469438, -0.3101935557860296, 0.1733113629107006, 0.08968593616209351, -0.1836344261367962, -0.06480023695256802, 0.18165070013886545, 0.19618275767992124, -0.07956460275570058, 0.0325997354656551  …  0.2845492418767769, 0.17406455870721682, -0.053101230371568706, -0.1382082812981627, 0.005830071475508364, 0.16739264037923055, 0.034365814374995335, 0.09107702398753297, -0.1877250428700409, 0.05116494897806923]
 [-0.04159442361185588, -0.7768029783272633, 0.06303616666722486, 0.08070518925253539, -0.07396265237309446, -0.06008109299719321, 0.07977141629715745, 0.019511027129056415, 0.08629917589924847, -0.11156298867318722  …  0.0792587504128044, -0.016444383900170008, -0.181746064577005, -0.01888129512990984, -0.13523922089388968, 0.11358102175659832, 0.07929049608459493, 0.1689565359083833, 0.07673657951723721, -0.1128480905648813]
 ⋮
 [-0.19830349374441875, -0.6086693423968884, 0.08552341811170468, 0.35781519334042255, 0.15790663648524367, 0.02712571268324985, 0.09855601327331667, -0.05840653973421127, -0.09546429767790429, -0.13414717696055448  …  -0.0430935804718714, 0.2678584478951765, 0.08780994289014614, 0.01613469379498457, 0.0516187906322884, -0.07383067566731401, -0.1481272738354552, -0.010532317187265649, 0.06555344745952187, -0.1506167863762911]
 [-0.04347524125197773, -0.6327981074196994, -0.221116680035191, 0.0282207467940456, -0.0855024881522933, 0.12821801740178346, 0.1779499563280024, -0.10247384887512365, 0.0396432464100116, -0.0582580338112627  …  0.1253893207083573, 0.09628202269764763, 0.3165295473947355, -0.14915034201394833, -0.1376727867817772, -0.004153096613530293, 0.09277957650773738, 0.05917264554031624, -0.12230262590034507, -0.19655728521529914]
 [-0.10173946348675116, -0.6475660153977272, 0.1260284619729566, -0.11933160462857616, -0.04774310633937567, 0.09093928358804217, 0.041662676324043114, -0.1264739543938265, 0.09605293126911392, -0.16790474428001648  …  -0.04056684573478108, 0.09351665120940456, 0.15259195558799882, 0.0009949298312580497, 0.09461980828206303, 0.3067004514287283, 0.16129258773733715, -0.18893664085007542, -0.1806865244492513, 0.029319680436405825]
 [-0.251780954320053, -0.39147463259941456, -0.24359579328578626, 0.30179309757665723, 0.21658893985206484, 0.12304585275893232, 0.28281133086451704, 0.029187615341955325, 0.03616243507191924, 0.029375588909979152  …  -0.08071746662465404, -0.2176101928258658, 0.20944684921170825, 0.043033273425352715, -0.040505542460853576, 0.17935596149079197, -0.08454569418519972, 0.0545941597033932, 0.12471741052450099, -0.24314124407858329]
 [0.28156471341150974, -0.6708572780452595, -0.1410302363738465, -0.08322589397277698, -0.022772599832907418, -0.04447265789199677, -0.016448068022011157, -0.07490911512503738, 0.2778432295769144, -0.10191899088372378  …  -0.057272155080983836, 0.12817478092201395, 0.04623814480781884, -0.12184190164369117, 0.1987855635987229, -0.14533603246124993, -0.16334072868597016, -0.052369977381939437, 0.014904286931394959, -0.2440882678882144]
 [0.12108727495744157, -0.714787344982596, 0.01632521838262752, 0.04437570556908449, -0.041199280304144284, 0.052984488452616, 0.03796520200156107, 0.2791785910964288, 0.11530429924056099, 0.12178223160398421  …  -0.07621847481721669, 0.18353870423743013, -0.19066653731436745, -0.09423224997242206, 0.14596847781388494, -0.09747986927777111, 0.16041150122587072, -0.02296513951256738, 0.06786878373578588, 0.15296635978447756]</pre>

<pre class='language-julia'><code class='language-julia'>F(M, y) = sum(1 / (2 * n) * distance.(Ref(M), Ref(y), data) .^ 2)</code></pre>
<pre class="code-output documenter-example-output" id="var-F">F (generic function with 1 method)</pre>

<pre class='language-julia'><code class='language-julia'>gradF(M, y) = sum(1 / n * grad_distance.(Ref(M), data, Ref(y)))</code></pre>
<pre class="code-output documenter-example-output" id="var-gradF">gradF (generic function with 1 method)</pre>


```
## Plain Examples
```@raw html
<div class="markdown">
<p>For the high level interfaces of the solvers, like <a href="https://manoptjl.org/stable/solvers/gradient_descent.html"><code>gradient_descent</code></a> we have to set <code>return_state</code> to <code>true</code> to obtain the whole options structure and not only the resulting minimizer.</p><p>Then we can easily use the <code>record=</code> option to add recorded values. This keyword accepts <code>RecordAction</code>s as well as several symbols as shortcuts, for example <code>:Cost</code> to record the cost, or if your options have a field <code>f</code>, <code>:f</code> would record that entry.</p></div>

<pre class='language-julia'><code class='language-julia'>R = gradient_descent(M, F, gradF, data[1]; record=:Cost, return_state=true)</code></pre>
<pre class="code-output documenter-example-output" id="var-R">RecordSolverState{DebugSolverState{GradientDescentState{Vector{Float64}, Vector{Float64}, StopWhenAny{Tuple{StopAfterIteration, StopWhenGradientNormLess}}, ArmijoLinesearch{ExponentialRetraction, Manopt.var"#58#61"}, ExponentialRetraction}}, NamedTuple{(:Iteration,), Tuple{RecordCost}}}(DebugSolverState{GradientDescentState{Vector{Float64}, Vector{Float64}, StopWhenAny{Tuple{StopAfterIteration, StopWhenGradientNormLess}}, ArmijoLinesearch{ExponentialRetraction, Manopt.var"#58#61"}, ExponentialRetraction}}(GradientDescentState{Vector{Float64}, Vector{Float64}, StopWhenAny{Tuple{StopAfterIteration, StopWhenGradientNormLess}}, ArmijoLinesearch{ExponentialRetraction, Manopt.var"#58#61"}, ExponentialRetraction}([0.003348564083448399, -0.9989177042553917, 0.01260395651475293, 3.724195907654294e-5, 0.00347688650874772, 0.007393312784223247, 0.0015131386301483914, -0.020957965200961437, 0.014862725158453362, -0.007213435087886661  …  -0.003341405141763289, 0.004841935454254808, -0.010592700175885091, -0.013017126493473067, 0.0033263346448692836, -0.004530004405886269, 0.0030077996849159536, -0.015610320483091028, -0.0016794415453862106, -0.002537572063969514], [-8.697873855618293e-12, -2.3160719388626728e-12, -1.2354431122539047e-11, 8.988731274876729e-11, -4.733831441395371e-11, -1.5449887164975876e-11, 8.260573948232642e-11, 9.896137755831605e-11, 1.5104898581919102e-10, -1.0275020011262306e-11  …  1.1865317292602574e-10, -9.099753063698465e-11, -1.1517346820179297e-10, 2.3269229384591424e-11, 7.209477754913964e-11, 5.79698539876234e-11, 6.894389193660283e-12, 5.861714630665208e-11, -1.2133124491003552e-10, -1.6233332061389716e-11], IdentityUpdateRule(), ArmijoLinesearch{ExponentialRetraction, Manopt.var"#58#61"}(1.0, ExponentialRetraction(), 0.95, 0.1, 1.758086915910625, 0.0, Manopt.var"#58#61"()), StopWhenAny{Tuple{StopAfterIteration, StopWhenGradientNormLess}}((StopAfterIteration(200, ""), StopWhenGradientNormLess(1.0e-9, "The algorithm reached approximately critical point after 60 iterations; the gradient norm (3.865928812772072e-10) is less than 1.0e-9.\n")), "The algorithm reached approximately critical point after 60 iterations; the gradient norm (3.865928812772072e-10) is less than 1.0e-9.\n"), ExponentialRetraction()), Dict{Symbol, Manopt.DebugAction}()), (Iteration = RecordCost([0.6868754085841271, 0.6240211444102515, 0.5900374782569906, 0.5691425134106758, 0.5512819383843195, 0.5421368100229834, 0.537458562738662, 0.5350045365259573, 0.5337243124406588, 0.5330491236590462  …  0.5322977905736729, 0.5322977905736708, 0.5322977905736694, 0.5322977905736688, 0.5322977905736685, 0.5322977905736683, 0.5322977905736681, 0.5322977905736681, 0.5322977905736681, 0.5322977905736681]),))</pre>


<div class="markdown"><p>From the returned options, we see that the <code>State</code> are encapsulated (decorated) with <code>RecordSolverState</code>.</p><p>You can attach different recorders to some operations (<code>:Start</code>. <code>:Stop</code>, <code>:Iteration</code> at time of  writing), where <code>:Iteration</code> is the default, so the following is the same as <code>get_record(R, :Iteation)</code>. We get</p></div>

<pre class='language-julia'><code class='language-julia'>get_record(R)</code></pre>
<pre class="code-output documenter-example-output" id="var-hash525965">60-element Vector{Float64}:
 0.6868754085841271
 0.6240211444102515
 0.5900374782569906
 0.5691425134106758
 0.5512819383843195
 0.5421368100229834
 0.537458562738662
 ⋮
 0.5322977905736685
 0.5322977905736683
 0.5322977905736681
 0.5322977905736681
 0.5322977905736681
 0.5322977905736681</pre>


<div class="markdown"><p>To record more than one value, you can pass an array of a mix of symbols and <code>RecordAction</code> which formally introduces <code>RecordGroup</code>. Such a group records a tuple of values in every iteration.</p></div>

<pre class='language-julia'><code class='language-julia'>R2 = gradient_descent(M, F, gradF, data[1]; record=[:Iteration, :Cost], return_state=true)</code></pre>
<pre class="code-output documenter-example-output" id="var-R2">RecordSolverState{DebugSolverState{GradientDescentState{Vector{Float64}, Vector{Float64}, StopWhenAny{Tuple{StopAfterIteration, StopWhenGradientNormLess}}, ArmijoLinesearch{ExponentialRetraction, Manopt.var"#58#61"}, ExponentialRetraction}}, NamedTuple{(:Iteration,), Tuple{RecordGroup}}}(DebugSolverState{GradientDescentState{Vector{Float64}, Vector{Float64}, StopWhenAny{Tuple{StopAfterIteration, StopWhenGradientNormLess}}, ArmijoLinesearch{ExponentialRetraction, Manopt.var"#58#61"}, ExponentialRetraction}}(GradientDescentState{Vector{Float64}, Vector{Float64}, StopWhenAny{Tuple{StopAfterIteration, StopWhenGradientNormLess}}, ArmijoLinesearch{ExponentialRetraction, Manopt.var"#58#61"}, ExponentialRetraction}([0.003348564083448399, -0.9989177042553917, 0.01260395651475293, 3.724195907654294e-5, 0.00347688650874772, 0.007393312784223247, 0.0015131386301483914, -0.020957965200961437, 0.014862725158453362, -0.007213435087886661  …  -0.003341405141763289, 0.004841935454254808, -0.010592700175885091, -0.013017126493473067, 0.0033263346448692836, -0.004530004405886269, 0.0030077996849159536, -0.015610320483091028, -0.0016794415453862106, -0.002537572063969514], [-8.697873855618293e-12, -2.3160719388626728e-12, -1.2354431122539047e-11, 8.988731274876729e-11, -4.733831441395371e-11, -1.5449887164975876e-11, 8.260573948232642e-11, 9.896137755831605e-11, 1.5104898581919102e-10, -1.0275020011262306e-11  …  1.1865317292602574e-10, -9.099753063698465e-11, -1.1517346820179297e-10, 2.3269229384591424e-11, 7.209477754913964e-11, 5.79698539876234e-11, 6.894389193660283e-12, 5.861714630665208e-11, -1.2133124491003552e-10, -1.6233332061389716e-11], IdentityUpdateRule(), ArmijoLinesearch{ExponentialRetraction, Manopt.var"#58#61"}(1.0, ExponentialRetraction(), 0.95, 0.1, 1.758086915910625, 0.0, Manopt.var"#58#61"()), StopWhenAny{Tuple{StopAfterIteration, StopWhenGradientNormLess}}((StopAfterIteration(200, ""), StopWhenGradientNormLess(1.0e-9, "The algorithm reached approximately critical point after 60 iterations; the gradient norm (3.865928812772072e-10) is less than 1.0e-9.\n")), "The algorithm reached approximately critical point after 60 iterations; the gradient norm (3.865928812772072e-10) is less than 1.0e-9.\n"), ExponentialRetraction()), Dict{Symbol, Manopt.DebugAction}()), (Iteration = RecordGroup(Manopt.RecordAction[RecordIteration([1, 2, 3, 4, 5, 6, 7, 8, 9, 10  …  51, 52, 53, 54, 55, 56, 57, 58, 59, 60]), RecordCost([0.6868754085841271, 0.6240211444102515, 0.5900374782569906, 0.5691425134106758, 0.5512819383843195, 0.5421368100229834, 0.537458562738662, 0.5350045365259573, 0.5337243124406588, 0.5330491236590462  …  0.5322977905736729, 0.5322977905736708, 0.5322977905736694, 0.5322977905736688, 0.5322977905736685, 0.5322977905736683, 0.5322977905736681, 0.5322977905736681, 0.5322977905736681, 0.5322977905736681])], Dict(:Iteration =&gt; 1, :Cost =&gt; 2)),))</pre>


<div class="markdown"><p>Here, the symbol <code>:Cost</code> is mapped to using the <code>RecordCost</code> action. The same holds for <code>:Iteration</code> and <code>:Iterate</code> and any member field of the current <code>State</code>. To access these you can first extract the group of records (that is where the <code>:Iteration</code>s are recorded – note the plural) and then access the <code>:Cost</code></p></div>

<pre class='language-julia'><code class='language-julia'>get_record_action(R2, :Iteration)</code></pre>
<pre class="code-output documenter-example-output" id="var-hash886786">RecordGroup(Manopt.RecordAction[RecordIteration([1, 2, 3, 4, 5, 6, 7, 8, 9, 10  …  51, 52, 53, 54, 55, 56, 57, 58, 59, 60]), RecordCost([0.6868754085841271, 0.6240211444102515, 0.5900374782569906, 0.5691425134106758, 0.5512819383843195, 0.5421368100229834, 0.537458562738662, 0.5350045365259573, 0.5337243124406588, 0.5330491236590462  …  0.5322977905736729, 0.5322977905736708, 0.5322977905736694, 0.5322977905736688, 0.5322977905736685, 0.5322977905736683, 0.5322977905736681, 0.5322977905736681, 0.5322977905736681, 0.5322977905736681])], Dict(:Iteration =&gt; 1, :Cost =&gt; 2))</pre>


<div class="markdown"><p><code>:Iteration</code> is the default here, i.e. something recorded through the iterations – and we can access the recorded data the same way as we specify them in the <code>record=</code> keyword, that is, using the indexing operation.</p></div>

<pre class='language-julia'><code class='language-julia'>get_record_action(R2)[:Cost]</code></pre>
<pre class="code-output documenter-example-output" id="var-hash178669">60-element Vector{Float64}:
 0.6868754085841271
 0.6240211444102515
 0.5900374782569906
 0.5691425134106758
 0.5512819383843195
 0.5421368100229834
 0.537458562738662
 ⋮
 0.5322977905736685
 0.5322977905736683
 0.5322977905736681
 0.5322977905736681
 0.5322977905736681
 0.5322977905736681</pre>


<div class="markdown"><p>This can be also done by using a the high level interface <code>get_record</code>.</p></div>

<pre class='language-julia'><code class='language-julia'>get_record(R2, :Iteration, :Cost)</code></pre>
<pre class="code-output documenter-example-output" id="var-hash112217">60-element Vector{Float64}:
 0.6868754085841271
 0.6240211444102515
 0.5900374782569906
 0.5691425134106758
 0.5512819383843195
 0.5421368100229834
 0.537458562738662
 ⋮
 0.5322977905736685
 0.5322977905736683
 0.5322977905736681
 0.5322977905736681
 0.5322977905736681
 0.5322977905736681</pre>


<div class="markdown"><p>Note that the first symbol again refers to the point where we record (not to the thing we record). We can also pass a tuple as second argument to have our own order (not that now the second <code>:Iteration</code> refers to the recorded iterations).</p></div>

<pre class='language-julia'><code class='language-julia'>get_record(R2, :Iteration, (:Iteration, :Cost))</code></pre>
<pre class="code-output documenter-example-output" id="var-hash123816">60-element Vector{Tuple{Int64, Float64}}:
 (1, 0.6868754085841271)
 (2, 0.6240211444102515)
 (3, 0.5900374782569906)
 (4, 0.5691425134106758)
 (5, 0.5512819383843195)
 (6, 0.5421368100229834)
 (7, 0.537458562738662)
 ⋮
 (55, 0.5322977905736685)
 (56, 0.5322977905736683)
 (57, 0.5322977905736681)
 (58, 0.5322977905736681)
 (59, 0.5322977905736681)
 (60, 0.5322977905736681)</pre>


```
## A more Complex Example
```@raw html
<div class="markdown">
<p>To illustrate a complicated example let's record:</p><ul><li><p>the iteration number, cost and gradient field, but only every sixth iteration;</p></li><li><p>the iteration at which we stop.</p></li></ul><p>We first generate the problem and the options, to also illustrate the low-level works when not using <code>gradient_descent</code>.</p></div>

<pre class='language-julia'><code class='language-julia'>p = DefaultManoptProblem(M, ManifoldGradientObjective(F, gradF))</code></pre>
<pre class="code-output documenter-example-output" id="var-p">DefaultManoptProblem{Sphere{30, ℝ}, ManifoldGradientObjective{AllocatingEvaluation, typeof(F), typeof(gradF)}}(Sphere(30, ℝ), ManifoldGradientObjective{AllocatingEvaluation, typeof(F), typeof(gradF)}(Main.var"workspace#3".F, Main.var"workspace#3".gradF))</pre>

<pre class='language-julia'><code class='language-julia'>o = GradientDescentState(
    M,
    copy(data[1]);
    stopping_criterion=StopAfterIteration(200) | StopWhenGradientNormLess(10.0^-9),
)</code></pre>
<pre class="code-output documenter-example-output" id="var-o">GradientDescentState{Vector{Float64}, Vector{Float64}, StopWhenAny{Tuple{StopAfterIteration, StopWhenGradientNormLess}}, ArmijoLinesearch{ExponentialRetraction, Manopt.var"#58#61"}, ExponentialRetraction}([-0.054658825167894595, -0.5592077846510423, -0.04738273828111257, -0.04682080720921302, 0.12279468849667038, 0.07171438895366239, -0.12930045409417057, -0.22102081626380404, -0.31805333254577767, 0.0065859500152017645  …  -0.21999168261518043, 0.19570142227077295, 0.340909965798364, -0.0310802190082894, -0.04674431076254687, -0.006088297671169996, 0.01576037011323387, -0.14523596850249543, 0.14526158060820338, 0.1972125856685378], [0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0  …  0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0], IdentityUpdateRule(), ArmijoLinesearch{ExponentialRetraction, Manopt.var"#58#61"}(1.0, ExponentialRetraction(), 0.95, 0.1, 1.0, 0.0, Manopt.var"#58#61"()), StopWhenAny{Tuple{StopAfterIteration, StopWhenGradientNormLess}}((StopAfterIteration(200, ""), StopWhenGradientNormLess(1.0e-9, "")), ""), ExponentialRetraction())</pre>


<div class="markdown"><p>We first build a <code>RecordGroup</code> to group the three entries we want to record per iteration. We then put this into a <code>RecordEvery</code> to only record this every 6th iteration</p></div>

<pre class='language-julia'><code class='language-julia'>rI = RecordEvery(
    RecordGroup([
        :Iteration =&gt; RecordIteration(),
        :Cost =&gt; RecordCost(),
        :Gradient =&gt; RecordEntry(similar(data[1]), :X),
    ]),
    6,
)</code></pre>
<pre class="code-output documenter-example-output" id="var-rI">RecordEvery(RecordGroup(Manopt.RecordAction[RecordIteration(Int64[]), RecordCost(Float64[]), RecordEntry{Vector{Float64}}(Vector{Float64}[], :X)], Dict(:Iteration =&gt; 1, :Gradient =&gt; 3, :Cost =&gt; 2)), 6, true)</pre>


<div class="markdown"><p>and a small option to record iterations</p></div>

<pre class='language-julia'><code class='language-julia'>sI = RecordIteration()</code></pre>
<pre class="code-output documenter-example-output" id="var-sI">RecordIteration(Int64[])</pre>


<div class="markdown"><p>We now combine both into the <code>RecordSolverState</code> decorator. It acts completely the same as an <code>Option</code> but records something in every iteration additionally. This is stored in a dictionary of <code>RecordActions</code>, where <code>:Iteration</code> is the action (here the only every 6th iteration group) and the <code>sI</code> which is executed at stop.</p><p>Note that the keyword <code>record=</code> (in the high level interface <code>gradient_descent</code> only would fill the <code>:Iteration</code> symbol).</p></div>

<pre class='language-julia'><code class='language-julia'>r = RecordSolverState(o, Dict(:Iteration =&gt; rI, :Stop =&gt; sI))</code></pre>
<pre class="code-output documenter-example-output" id="var-r">RecordSolverState{GradientDescentState{Vector{Float64}, Vector{Float64}, StopWhenAny{Tuple{StopAfterIteration, StopWhenGradientNormLess}}, ArmijoLinesearch{ExponentialRetraction, Manopt.var"#58#61"}, ExponentialRetraction}, NamedTuple{(:Iteration, :Stop), Tuple{RecordEvery, RecordIteration}}}(GradientDescentState{Vector{Float64}, Vector{Float64}, StopWhenAny{Tuple{StopAfterIteration, StopWhenGradientNormLess}}, ArmijoLinesearch{ExponentialRetraction, Manopt.var"#58#61"}, ExponentialRetraction}([-0.054658825167894595, -0.5592077846510423, -0.04738273828111257, -0.04682080720921302, 0.12279468849667038, 0.07171438895366239, -0.12930045409417057, -0.22102081626380404, -0.31805333254577767, 0.0065859500152017645  …  -0.21999168261518043, 0.19570142227077295, 0.340909965798364, -0.0310802190082894, -0.04674431076254687, -0.006088297671169996, 0.01576037011323387, -0.14523596850249543, 0.14526158060820338, 0.1972125856685378], [0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0  …  0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0], IdentityUpdateRule(), ArmijoLinesearch{ExponentialRetraction, Manopt.var"#58#61"}(1.0, ExponentialRetraction(), 0.95, 0.1, 1.0, 0.0, Manopt.var"#58#61"()), StopWhenAny{Tuple{StopAfterIteration, StopWhenGradientNormLess}}((StopAfterIteration(200, ""), StopWhenGradientNormLess(1.0e-9, "")), ""), ExponentialRetraction()), (Iteration = RecordEvery(RecordGroup(Manopt.RecordAction[RecordIteration(Int64[]), RecordCost(Float64[]), RecordEntry{Vector{Float64}}(Vector{Float64}[], :X)], Dict(:Iteration =&gt; 1, :Gradient =&gt; 3, :Cost =&gt; 2)), 6, true), Stop = RecordIteration(Int64[])))</pre>


<div class="markdown"><p>We now call the solver</p></div>

<pre class='language-julia'><code class='language-julia'>res = solve!(p, r)</code></pre>
<pre class="code-output documenter-example-output" id="var-res">RecordSolverState{GradientDescentState{Vector{Float64}, Vector{Float64}, StopWhenAny{Tuple{StopAfterIteration, StopWhenGradientNormLess}}, ArmijoLinesearch{ExponentialRetraction, Manopt.var"#58#61"}, ExponentialRetraction}, NamedTuple{(:Iteration, :Stop), Tuple{RecordEvery, RecordIteration}}}(GradientDescentState{Vector{Float64}, Vector{Float64}, StopWhenAny{Tuple{StopAfterIteration, StopWhenGradientNormLess}}, ArmijoLinesearch{ExponentialRetraction, Manopt.var"#58#61"}, ExponentialRetraction}([0.003348564083448399, -0.9989177042553917, 0.01260395651475293, 3.724195907654294e-5, 0.00347688650874772, 0.007393312784223247, 0.0015131386301483914, -0.020957965200961437, 0.014862725158453362, -0.007213435087886661  …  -0.003341405141763289, 0.004841935454254808, -0.010592700175885091, -0.013017126493473067, 0.0033263346448692836, -0.004530004405886269, 0.0030077996849159536, -0.015610320483091028, -0.0016794415453862106, -0.002537572063969514], [-8.697873855618293e-12, -2.3160719388626728e-12, -1.2354431122539047e-11, 8.988731274876729e-11, -4.733831441395371e-11, -1.5449887164975876e-11, 8.260573948232642e-11, 9.896137755831605e-11, 1.5104898581919102e-10, -1.0275020011262306e-11  …  1.1865317292602574e-10, -9.099753063698465e-11, -1.1517346820179297e-10, 2.3269229384591424e-11, 7.209477754913964e-11, 5.79698539876234e-11, 6.894389193660283e-12, 5.861714630665208e-11, -1.2133124491003552e-10, -1.6233332061389716e-11], IdentityUpdateRule(), ArmijoLinesearch{ExponentialRetraction, Manopt.var"#58#61"}(1.0, ExponentialRetraction(), 0.95, 0.1, 1.758086915910625, 0.0, Manopt.var"#58#61"()), StopWhenAny{Tuple{StopAfterIteration, StopWhenGradientNormLess}}((StopAfterIteration(200, ""), StopWhenGradientNormLess(1.0e-9, "The algorithm reached approximately critical point after 60 iterations; the gradient norm (3.865928812772072e-10) is less than 1.0e-9.\n")), "The algorithm reached approximately critical point after 60 iterations; the gradient norm (3.865928812772072e-10) is less than 1.0e-9.\n"), ExponentialRetraction()), (Iteration = RecordEvery(RecordGroup(Manopt.RecordAction[RecordIteration([6, 12, 18, 24, 30, 36, 42, 48, 54, 60]), RecordCost([0.5421368100229834, 0.5325071127227714, 0.532302375710409, 0.5322978928223222, 0.5322977928970517, 0.5322977906274986, 0.53229779057494, 0.5322977905736986, 0.5322977905736688, 0.5322977905736681]), RecordEntry{Vector{Float64}}([[0.00915953309137745, 0.03636198060300825, 0.00876918980030283, 0.00993021269193873, -0.022412050287073305, -0.012383003092273357, 0.024240090364743658, 0.0394002573586919, 0.059002094743271, -0.0015645432344094934  …  0.041039917981609095, -0.03559037560054106, -0.06083036128387102, 0.004541947339369046, 0.009377692584584511, 0.0018449994535462736, -0.002533013401769356, 0.02517978082666557, -0.028158761621277784, -0.033710429737238964], [0.0012076448685088474, 0.0006247496221492019, 0.001234435420042108, 0.0018297123706428578, -0.003414910518214241, -0.001774365390798262, 0.0038034535193429897, 0.005943389589094038, 0.009039716155376515, -0.00027813581976868115  …  0.0064105743566829, -0.005401551609299228, -0.008933208089699884, 0.0005968336160260089, 0.0016211478670105019, 0.0004943190638360105, -0.0003127378316082481, 0.00368203429481012, -0.004598417514115335, -0.004679951023424969], [0.00015069891133301318, -7.536540048538633e-6, 0.00015937754737447703, 0.00032627100535932523, -0.0005115633746957628, -0.0002527985568482627, 0.0005879698315850519, 0.0008966104539735962, 0.0013555664186435854, -4.3551120859806826e-5  …  0.0009854459609708076, -0.000811267519312202, -0.0012882632609157267, 8.78726305238526e-5, 0.00027512849454913474, 0.00011294625339284032, -3.683823079805271e-5, 0.0005418159425187017, -0.0007369261137125317, -0.0006329705091328134], [1.805360615506684e-5, -3.3078788343558515e-6, 1.957230672267371e-5, 5.739678313633607e-5, -7.648524150156486e-5, -3.5965217709055494e-5, 9.12645719995581e-5, 0.00013587163331824006, 0.00020386997581573694, -6.821636765625723e-6  …  0.00015151210105835114, -0.00012235888575716047, -0.00018667045587536957, 1.3814317174037035e-5, 4.688659791442248e-5, 2.347088099899727e-5, -3.939988088165356e-6, 8.047255063359164e-5, -0.00011813470334849589, -8.499242433160351e-5], [2.0287764934330034e-6, -5.415136805044474e-7, 2.2335655191911568e-6, 9.996148295219573e-6, -1.1417012733191904e-5, -5.101654603301148e-6, 1.4229883450351539e-5, 2.0646020770081217e-5, 3.079343486410703e-5, -1.0898281254483045e-6  …  2.3309186739861465e-5, -1.853060668838055e-5, -2.721787798694791e-5, 2.2765348495770656e-6, 8.014997411630808e-6, 4.610768270932303e-6, -3.348907692544197e-7, 1.204329960988763e-5, -1.8953463841046267e-5, -1.1313190991686108e-5], [2.0212763877255316e-7, -8.192214853691162e-8, 2.2158993240852318e-7, 1.7275000758433843e-6, -1.7020498919837921e-6, -7.215058313153193e-7, 2.228532052411785e-6, 3.1439778726712313e-6, 4.672728118251102e-6, -1.7856791995892818e-7  …  3.589118964370631e-6, -2.817763139359767e-6, -3.995825686360448e-6, 3.87913302207985e-7, 1.3720757437822042e-6, 8.734527700877829e-7, -8.57419083592748e-9, 1.8156381881770333e-6, -3.0434330996960987e-6, -1.4884848693356642e-6], [1.4804523557721905e-8, -1.2264513258145352e-8, 1.4900474345406781e-8, 2.9674129945271544e-7, -2.535106862279783e-7, -1.0176862634359376e-7, 3.505158506873833e-7, 4.796365395957252e-7, 7.124565981893956e-7, -3.003444344328054e-8  …  5.532672990073839e-7, -4.302056101730331e-7, -5.909513064490538e-7, 6.750616783138806e-8, 2.3492870135689045e-7, 1.613458018314654e-7, 5.517408280876794e-9, 2.757191364454001e-7, -4.891061063070105e-7, -1.9276282473964178e-7], [-1.568041416654363e-10, -1.8334957477909503e-9, -7.318143797945882e-10, 5.0729753610175466e-8, -3.773815549514253e-8, -1.432429260038589e-8, 5.5362649100126774e-8, 7.328422851958331e-8, 1.0915931142457531e-7, -5.174492532643727e-9  …  8.539972284198093e-8, -6.594753330886128e-8, -8.808095039740307e-8, 1.1879923997906413e-8, 4.019913582076814e-8, 2.9260115344000547e-8, 1.9422224188366064e-9, 4.21708145098321e-8, -7.867004047687313e-8, -2.441377067118034e-8], [-3.9120571417180193e-10, -2.7392009367598817e-10, -5.901014368445666e-10, 8.639579300264615e-9, -5.616689558574474e-9, -2.0135816242600755e-9, 8.779963869217668e-9, 1.1211385624780926e-8, 1.680707340857325e-8, -9.093571166798702e-10  …  1.3201483553597565e-8, -1.0149867185171423e-8, -1.323598786243151e-8, 2.0995057312009303e-9, 6.870561061186409e-9, 5.23265641626529e-9, 4.744203675472857e-10, 6.495168518533354e-9, -1.266449198551007e-8, -2.9926285735799602e-9], [-8.697873855618293e-12, -2.3160719388626728e-12, -1.2354431122539047e-11, 8.988731274876729e-11, -4.733831441395371e-11, -1.5449887164975876e-11, 8.260573948232642e-11, 9.896137755831605e-11, 1.5104898581919102e-10, -1.0275020011262306e-11  …  1.1865317292602574e-10, -9.099753063698465e-11, -1.1517346820179297e-10, 2.3269229384591424e-11, 7.209477754913964e-11, 5.79698539876234e-11, 6.894389193660283e-12, 5.861714630665208e-11, -1.2133124491003552e-10, -1.6233332061389716e-11]], :X)], Dict(:Iteration =&gt; 1, :Gradient =&gt; 3, :Cost =&gt; 2)), 6, true), Stop = RecordIteration([60])))</pre>


<div class="markdown"><p>And we can check the recorded value at <code>:Stop</code> to see how many iterations were performed</p></div>

<pre class='language-julia'><code class='language-julia'>get_record(res, :Stop)</code></pre>
<pre class="code-output documenter-example-output" id="var-hash654462">1-element Vector{Int64}:
 60</pre>


<div class="markdown"><p>and the other values during the iterations are</p></div>

<pre class='language-julia'><code class='language-julia'>get_record(res, :Iteration, (:Iteration, :Cost))</code></pre>
<pre class="code-output documenter-example-output" id="var-hash619888">10-element Vector{Tuple{Int64, Float64}}:
 (6, 0.5421368100229834)
 (12, 0.5325071127227714)
 (18, 0.532302375710409)
 (24, 0.5322978928223222)
 (30, 0.5322977928970517)
 (36, 0.5322977906274986)
 (42, 0.53229779057494)
 (48, 0.5322977905736986)
 (54, 0.5322977905736688)
 (60, 0.5322977905736681)</pre>


<div class="markdown"><h2>Writing an own <code>RecordAction</code>s</h2><p>Let's investigate where we want to count the number of function evaluations, again just to illustrate, since for the gradient this is just one evaluation per iteration. We first define a cost, that counts its own calls.</p></div>

<pre class='language-julia'><code class='language-julia'>begin
    mutable struct MyCost{T}
        data::T
        count::Int
    end
    MyCost(data::T) where {T} = MyCost{T}(data, 0)
    function (c::MyCost)(M, x)
        c.count += 1
        return sum(1 / (2 * length(c.data)) * distance.(Ref(M), Ref(x), c.data) .^ 2)
    end
end</code></pre>



<div class="markdown"><p>and we define the following RecordAction, which is a functor, i.e. a struct that is also a function. The function we have to implement is similar to a single solver step in signature, since it might get called every iteration:</p></div>

<pre class='language-julia'><code class='language-julia'>begin
    mutable struct RecordCount &lt;: RecordAction
        recorded_values::Vector{Int}
        RecordCount() = new(Vector{Int}())
    end
    function (r::RecordCount)(p::AbstractManoptProblem, ::AbstractManoptSolverState, i)
        if i &gt; 0
            push!(r.recorded_values, get_cost_function(get_objective(p)).count)
        elseif i &lt; 0 # reset if negative
            r.recorded_values = Vector{Int}()
        end
    end
end</code></pre>



<div class="markdown"><p>Now we can initialize the new cost and call the gradient descent. Note that this illustrates also the last use case – you can pass symbol-action pairs into the <code>record=</code>array.</p></div>

<pre class='language-julia'><code class='language-julia'>F2 = MyCost(data)</code></pre>
<pre class="code-output documenter-example-output" id="var-F2">MyCost{Vector{Vector{Float64}}}([[-0.054658825167894595, -0.5592077846510423, -0.04738273828111257, -0.04682080720921302, 0.12279468849667038, 0.07171438895366239, -0.12930045409417057, -0.22102081626380404, -0.31805333254577767, 0.0065859500152017645  …  -0.21999168261518043, 0.19570142227077295, 0.340909965798364, -0.0310802190082894, -0.04674431076254687, -0.006088297671169996, 0.01576037011323387, -0.14523596850249543, 0.14526158060820338, 0.1972125856685378], [-0.08192376929745249, -0.5097715132187676, -0.008339904915541005, 0.07289741328038676, 0.11422036270613797, -0.11546739299835748, 0.2296996932628472, 0.1490467170835958, -0.11124820565850364, -0.11790721606521781  …  -0.16421249630470344, -0.2450575844467715, -0.07570080850379841, -0.07426218324072491, -0.026520181327346338, 0.11555341205250205, -0.0292955762365121, -0.09012096853677576, -0.23470556634911574, -0.026214242996704013], [-0.22951484264859257, -0.6083825348640186, 0.14273766477054015, -0.11947823367023377, 0.05984293499234536, 0.058820835498203126, 0.07577331705863266, 0.1632847202946857, 0.20244385489915745, 0.04389826920203656  …  0.3222365119325929, 0.009728730325524067, -0.12094785371632395, -0.36322323926212824, -0.0689253407939657, 0.23356953371702974, 0.23489531397909744, 0.078303336494718, -0.14272984135578806, 0.07844539956202407], [-0.0012588500237817606, -0.29958740415089763, 0.036738459489123514, 0.20567651907595125, -0.1131046432541904, -0.06032435985370224, 0.3366633723165895, -0.1694687746143405, -0.001987171245125281, 0.04933779858684409  …  -0.2399584473006256, 0.19889267065775063, 0.22468755918787048, 0.1780090580180643, 0.023703860700539356, -0.10212737517121755, 0.03807004103115319, -0.20569120952458983, -0.03257704254233959, 0.06925473452536687], [-0.035534309946938375, -0.06645560787329002, 0.14823972268208874, -0.23913346587232426, 0.038347027875883496, 0.10453333143286662, 0.050933995140290705, -0.12319549375687473, 0.12956684644537844, -0.23540367869989412  …  -0.41471772859912864, -0.1418984610380257, 0.0038321446836859334, 0.23655566917750157, -0.17500681300994742, -0.039189751036839374, -0.08687860620942896, -0.11509948162959047, 0.11378233994840942, 0.38739450723013735], [-0.3122539912469438, -0.3101935557860296, 0.1733113629107006, 0.08968593616209351, -0.1836344261367962, -0.06480023695256802, 0.18165070013886545, 0.19618275767992124, -0.07956460275570058, 0.0325997354656551  …  0.2845492418767769, 0.17406455870721682, -0.053101230371568706, -0.1382082812981627, 0.005830071475508364, 0.16739264037923055, 0.034365814374995335, 0.09107702398753297, -0.1877250428700409, 0.05116494897806923], [-0.04159442361185588, -0.7768029783272633, 0.06303616666722486, 0.08070518925253539, -0.07396265237309446, -0.06008109299719321, 0.07977141629715745, 0.019511027129056415, 0.08629917589924847, -0.11156298867318722  …  0.0792587504128044, -0.016444383900170008, -0.181746064577005, -0.01888129512990984, -0.13523922089388968, 0.11358102175659832, 0.07929049608459493, 0.1689565359083833, 0.07673657951723721, -0.1128480905648813], [-0.21221814304651335, -0.5031823821503253, 0.010326342133992458, -0.12438192100961257, 0.04004758695231872, 0.2280527500843805, -0.2096243232022162, -0.16564828762420294, -0.28325749481138984, 0.17033534605245823  …  -0.13599096505924074, 0.28437770540525625, 0.08424426798544583, -0.1266207606984139, 0.04917635557603396, -0.00012608938533809706, -0.04283220254770056, -0.08771365647566572, 0.14750169103093985, 0.11601120086036351], [0.10683290707435536, -0.17680836277740156, 0.23767458301899405, 0.12011180867097299, -0.029404774462600154, 0.11522028383799933, -0.3318174480974519, -0.17859266746938374, 0.04352373642537759, 0.2530382802667988  …  0.08879861736692073, -0.004412506987801729, 0.19786810509925895, -0.1397104682727044, 0.09482328498485094, 0.05108149065160893, -0.14578343506951633, 0.3167479772660438, 0.10422673169182732, 0.21573150015891313], [-0.024895624707466164, -0.7473912016432697, -0.1392537238944721, -0.14948896791465557, -0.09765393283580377, 0.04413059403279867, -0.13865379004720355, -0.071032040283992, 0.15604054722246585, -0.10744260463413555  …  -0.14748067081342833, -0.14743635071251024, 0.0643591937981352, 0.16138827697852615, -0.12656652133603935, -0.06463635704869083, 0.14329582429103488, -0.01113113793821713, 0.29295387893749997, 0.06774523575259782]  …  [0.011874845316569967, -0.6910596618389588, 0.21275741439477827, -0.014042545524367437, -0.07883613103495014, -0.0021900966696246776, -0.033836430464220496, 0.2925813113264835, -0.04718187201980008, 0.03949680289730036  …  0.0867736586603294, 0.0404682510051544, -0.24779813848587257, -0.28631514602877145, -0.07211767532456789, -0.15072898498180473, 0.017855923621826746, -0.09795357710255254, -0.14755229203084924, 0.1305005778855436], [0.013457629515450426, -0.3750353654626534, 0.12349883726772073, 0.3521803555005319, 0.2475921439420274, 0.006088649842999206, 0.31203183112392907, -0.036869203979483754, -0.07475746464056504, -0.029297797064479717  …  0.16867368684091563, -0.09450564983271922, -0.0587273302122711, -0.1326667940553803, -0.25530237980444614, 0.37556905374043376, 0.04922612067677609, 0.2605362549983866, -0.21871556587505667, -0.22915883767386164], [0.03295085436260177, -0.971861604433394, 0.034748713521512035, -0.0494065013245799, -0.01767479281403355, 0.0465459739459587, 0.007470494722096038, 0.003227960072276129, 0.0058328596338402365, -0.037591237446692356  …  0.03205152122876297, 0.11331109854742015, 0.03044900529526686, 0.017971704993311105, -0.009329252062960229, -0.02939354719650879, 0.022088835776251863, -0.02546111553658854, -0.0026257225461427582, 0.005702111697172774], [0.06968243992532257, -0.7119502191435176, -0.18136614593117445, -0.1695926215673451, 0.01725015359973796, -0.00694164951158388, -0.34621134287344574, 0.024709256792651912, -0.1632255805999673, -0.2158226433583082  …  -0.14153772108081458, -0.11256850346909901, 0.045109821764180706, -0.1162754336222613, -0.13221711766357983, 0.005365354776191061, 0.012750671705879105, -0.018208207549835407, 0.12458753932455452, -0.31843587960340897], [-0.19830349374441875, -0.6086693423968884, 0.08552341811170468, 0.35781519334042255, 0.15790663648524367, 0.02712571268324985, 0.09855601327331667, -0.05840653973421127, -0.09546429767790429, -0.13414717696055448  …  -0.0430935804718714, 0.2678584478951765, 0.08780994289014614, 0.01613469379498457, 0.0516187906322884, -0.07383067566731401, -0.1481272738354552, -0.010532317187265649, 0.06555344745952187, -0.1506167863762911], [-0.04347524125197773, -0.6327981074196994, -0.221116680035191, 0.0282207467940456, -0.0855024881522933, 0.12821801740178346, 0.1779499563280024, -0.10247384887512365, 0.0396432464100116, -0.0582580338112627  …  0.1253893207083573, 0.09628202269764763, 0.3165295473947355, -0.14915034201394833, -0.1376727867817772, -0.004153096613530293, 0.09277957650773738, 0.05917264554031624, -0.12230262590034507, -0.19655728521529914], [-0.10173946348675116, -0.6475660153977272, 0.1260284619729566, -0.11933160462857616, -0.04774310633937567, 0.09093928358804217, 0.041662676324043114, -0.1264739543938265, 0.09605293126911392, -0.16790474428001648  …  -0.04056684573478108, 0.09351665120940456, 0.15259195558799882, 0.0009949298312580497, 0.09461980828206303, 0.3067004514287283, 0.16129258773733715, -0.18893664085007542, -0.1806865244492513, 0.029319680436405825], [-0.251780954320053, -0.39147463259941456, -0.24359579328578626, 0.30179309757665723, 0.21658893985206484, 0.12304585275893232, 0.28281133086451704, 0.029187615341955325, 0.03616243507191924, 0.029375588909979152  …  -0.08071746662465404, -0.2176101928258658, 0.20944684921170825, 0.043033273425352715, -0.040505542460853576, 0.17935596149079197, -0.08454569418519972, 0.0545941597033932, 0.12471741052450099, -0.24314124407858329], [0.28156471341150974, -0.6708572780452595, -0.1410302363738465, -0.08322589397277698, -0.022772599832907418, -0.04447265789199677, -0.016448068022011157, -0.07490911512503738, 0.2778432295769144, -0.10191899088372378  …  -0.057272155080983836, 0.12817478092201395, 0.04623814480781884, -0.12184190164369117, 0.1987855635987229, -0.14533603246124993, -0.16334072868597016, -0.052369977381939437, 0.014904286931394959, -0.2440882678882144], [0.12108727495744157, -0.714787344982596, 0.01632521838262752, 0.04437570556908449, -0.041199280304144284, 0.052984488452616, 0.03796520200156107, 0.2791785910964288, 0.11530429924056099, 0.12178223160398421  …  -0.07621847481721669, 0.18353870423743013, -0.19066653731436745, -0.09423224997242206, 0.14596847781388494, -0.09747986927777111, 0.16041150122587072, -0.02296513951256738, 0.06786878373578588, 0.15296635978447756]], 0)</pre>


<div class="markdown"><p>Now for the plain gradient descent, we have to modify the step (to a constant stepsize) and remove the default check whether the cost increases (setting <code>debug</code> to <code>[]</code>). We also only look at the first 20 iterations to keep this example small in recorded values. We call</p></div>

<pre class='language-julia'><code class='language-julia'>R3 = gradient_descent(
    M,
    F2,
    gradF,
    data[1];
    record=[:Iteration, :Count =&gt; RecordCount(), :Cost],
    stepsize = ConstantStepsize(1.0),
    stopping_criterion=StopAfterIteration(20),
    debug=[],
    return_state=true,
)</code></pre>
<pre class="code-output documenter-example-output" id="var-R3">RecordSolverState{DebugSolverState{GradientDescentState{Vector{Float64}, Vector{Float64}, StopAfterIteration, ConstantStepsize{Float64}, ExponentialRetraction}}, NamedTuple{(:Iteration,), Tuple{RecordGroup}}}(DebugSolverState{GradientDescentState{Vector{Float64}, Vector{Float64}, StopAfterIteration, ConstantStepsize{Float64}, ExponentialRetraction}}(GradientDescentState{Vector{Float64}, Vector{Float64}, StopAfterIteration, ConstantStepsize{Float64}, ExponentialRetraction}([0.003348563697666394, -0.9989177042237649, 0.012603956176158473, 3.724185091495233e-5, 0.0034768871059143187, 0.00739331315108791, 0.001513138004540016, -0.020957966249101206, 0.014862723464370162, -0.00721343503596234  …  -0.0033414061638458135, 0.004841936440829279, -0.010592698188325094, -0.013017126662754641, 0.0033263344419417022, -0.004530004343370803, 0.003007799776176174, -0.015610321221756977, -0.00167944091503655, -0.0025375708198013126], [-6.129711312369366e-10, 5.181299264501706e-11, -5.405597904838561e-10, -2.1340247261360593e-10, 9.87424036788954e-10, 5.979410718865092e-10, -1.042001310519599e-9, -1.7296361457344069e-9, -2.7893922159864717e-9, 8.807665393458647e-11  …  -1.7041825297941723e-9, 1.6257109105633806e-9, 3.2321891115720362e-9, -2.7365968852436327e-10, -3.5264198587061854e-10, 7.746778731194427e-11, 1.4421732013355885e-10, -1.2080581333427773e-9, 1.0696541396360395e-9, 1.9981575832145066e-9], IdentityUpdateRule(), ConstantStepsize{Float64}(1.0), StopAfterIteration(20, "The algorithm reached its maximal number of iterations (20).\n"), ExponentialRetraction()), Dict{Symbol, Manopt.DebugAction}()), (Iteration = RecordGroup(Manopt.RecordAction[RecordIteration([1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20]), RecordCount([0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19]), RecordCost([0.5808287253777763, 0.5395268557323747, 0.5333529073733115, 0.5324514620174544, 0.532320174366715, 0.5323010518577254, 0.5322982658416164, 0.532297859847447, 0.5322978006725338, 0.5322977920461376, 0.5322977907883958, 0.5322977906049866, 0.5322977905782368, 0.532297790574335, 0.5322977905737658, 0.5322977905736826, 0.5322977905736704, 0.5322977905736687, 0.5322977905736683, 0.5322977905736681])], Dict(:Iteration =&gt; 1, :Count =&gt; 2, :Cost =&gt; 3)),))</pre>


<div class="markdown"><p>For <code>:Cost</code> we already learned how to access them, the <code>:Count =&gt;</code> introduces the following action to obtain the <code>:Count</code>. We can again access the whole sets of records</p></div>

<pre class='language-julia'><code class='language-julia'>get_record(R3)</code></pre>
<pre class="code-output documenter-example-output" id="var-hash558608">20-element Vector{Tuple{Int64, Int64, Float64}}:
 (1, 0, 0.5808287253777763)
 (2, 1, 0.5395268557323747)
 (3, 2, 0.5333529073733115)
 (4, 3, 0.5324514620174544)
 (5, 4, 0.532320174366715)
 (6, 5, 0.5323010518577254)
 (7, 6, 0.5322982658416164)
 ⋮
 (15, 14, 0.5322977905737658)
 (16, 15, 0.5322977905736826)
 (17, 16, 0.5322977905736704)
 (18, 17, 0.5322977905736687)
 (19, 18, 0.5322977905736683)
 (20, 19, 0.5322977905736681)</pre>


<div class="markdown"><p>this is equivalent to calling <code>R[:Iteration]</code>. Note that since we introduced <code>:Count</code> we can also access a single recorded value using</p></div>

<pre class='language-julia'><code class='language-julia'>R3[:Iteration, :Count]</code></pre>
<pre class="code-output documenter-example-output" id="var-hash917209">20-element Vector{Int64}:
  0
  1
  2
  3
  4
  5
  6
  ⋮
 14
 15
 16
 17
 18
 19</pre>


<div class="markdown"><p>and we see that the cost function is called once per iteration.</p></div>


<div class="markdown"><p>If we use this counting cost and run the default gradient descent with Armijo linesearch, we can infer how many Armijo linesearch backtracks are preformed:</p></div>

<pre class='language-julia'><code class='language-julia'>F3 = MyCost(data)</code></pre>
<pre class="code-output documenter-example-output" id="var-F3">MyCost{Vector{Vector{Float64}}}([[-0.054658825167894595, -0.5592077846510423, -0.04738273828111257, -0.04682080720921302, 0.12279468849667038, 0.07171438895366239, -0.12930045409417057, -0.22102081626380404, -0.31805333254577767, 0.0065859500152017645  …  -0.21999168261518043, 0.19570142227077295, 0.340909965798364, -0.0310802190082894, -0.04674431076254687, -0.006088297671169996, 0.01576037011323387, -0.14523596850249543, 0.14526158060820338, 0.1972125856685378], [-0.08192376929745249, -0.5097715132187676, -0.008339904915541005, 0.07289741328038676, 0.11422036270613797, -0.11546739299835748, 0.2296996932628472, 0.1490467170835958, -0.11124820565850364, -0.11790721606521781  …  -0.16421249630470344, -0.2450575844467715, -0.07570080850379841, -0.07426218324072491, -0.026520181327346338, 0.11555341205250205, -0.0292955762365121, -0.09012096853677576, -0.23470556634911574, -0.026214242996704013], [-0.22951484264859257, -0.6083825348640186, 0.14273766477054015, -0.11947823367023377, 0.05984293499234536, 0.058820835498203126, 0.07577331705863266, 0.1632847202946857, 0.20244385489915745, 0.04389826920203656  …  0.3222365119325929, 0.009728730325524067, -0.12094785371632395, -0.36322323926212824, -0.0689253407939657, 0.23356953371702974, 0.23489531397909744, 0.078303336494718, -0.14272984135578806, 0.07844539956202407], [-0.0012588500237817606, -0.29958740415089763, 0.036738459489123514, 0.20567651907595125, -0.1131046432541904, -0.06032435985370224, 0.3366633723165895, -0.1694687746143405, -0.001987171245125281, 0.04933779858684409  …  -0.2399584473006256, 0.19889267065775063, 0.22468755918787048, 0.1780090580180643, 0.023703860700539356, -0.10212737517121755, 0.03807004103115319, -0.20569120952458983, -0.03257704254233959, 0.06925473452536687], [-0.035534309946938375, -0.06645560787329002, 0.14823972268208874, -0.23913346587232426, 0.038347027875883496, 0.10453333143286662, 0.050933995140290705, -0.12319549375687473, 0.12956684644537844, -0.23540367869989412  …  -0.41471772859912864, -0.1418984610380257, 0.0038321446836859334, 0.23655566917750157, -0.17500681300994742, -0.039189751036839374, -0.08687860620942896, -0.11509948162959047, 0.11378233994840942, 0.38739450723013735], [-0.3122539912469438, -0.3101935557860296, 0.1733113629107006, 0.08968593616209351, -0.1836344261367962, -0.06480023695256802, 0.18165070013886545, 0.19618275767992124, -0.07956460275570058, 0.0325997354656551  …  0.2845492418767769, 0.17406455870721682, -0.053101230371568706, -0.1382082812981627, 0.005830071475508364, 0.16739264037923055, 0.034365814374995335, 0.09107702398753297, -0.1877250428700409, 0.05116494897806923], [-0.04159442361185588, -0.7768029783272633, 0.06303616666722486, 0.08070518925253539, -0.07396265237309446, -0.06008109299719321, 0.07977141629715745, 0.019511027129056415, 0.08629917589924847, -0.11156298867318722  …  0.0792587504128044, -0.016444383900170008, -0.181746064577005, -0.01888129512990984, -0.13523922089388968, 0.11358102175659832, 0.07929049608459493, 0.1689565359083833, 0.07673657951723721, -0.1128480905648813], [-0.21221814304651335, -0.5031823821503253, 0.010326342133992458, -0.12438192100961257, 0.04004758695231872, 0.2280527500843805, -0.2096243232022162, -0.16564828762420294, -0.28325749481138984, 0.17033534605245823  …  -0.13599096505924074, 0.28437770540525625, 0.08424426798544583, -0.1266207606984139, 0.04917635557603396, -0.00012608938533809706, -0.04283220254770056, -0.08771365647566572, 0.14750169103093985, 0.11601120086036351], [0.10683290707435536, -0.17680836277740156, 0.23767458301899405, 0.12011180867097299, -0.029404774462600154, 0.11522028383799933, -0.3318174480974519, -0.17859266746938374, 0.04352373642537759, 0.2530382802667988  …  0.08879861736692073, -0.004412506987801729, 0.19786810509925895, -0.1397104682727044, 0.09482328498485094, 0.05108149065160893, -0.14578343506951633, 0.3167479772660438, 0.10422673169182732, 0.21573150015891313], [-0.024895624707466164, -0.7473912016432697, -0.1392537238944721, -0.14948896791465557, -0.09765393283580377, 0.04413059403279867, -0.13865379004720355, -0.071032040283992, 0.15604054722246585, -0.10744260463413555  …  -0.14748067081342833, -0.14743635071251024, 0.0643591937981352, 0.16138827697852615, -0.12656652133603935, -0.06463635704869083, 0.14329582429103488, -0.01113113793821713, 0.29295387893749997, 0.06774523575259782]  …  [0.011874845316569967, -0.6910596618389588, 0.21275741439477827, -0.014042545524367437, -0.07883613103495014, -0.0021900966696246776, -0.033836430464220496, 0.2925813113264835, -0.04718187201980008, 0.03949680289730036  …  0.0867736586603294, 0.0404682510051544, -0.24779813848587257, -0.28631514602877145, -0.07211767532456789, -0.15072898498180473, 0.017855923621826746, -0.09795357710255254, -0.14755229203084924, 0.1305005778855436], [0.013457629515450426, -0.3750353654626534, 0.12349883726772073, 0.3521803555005319, 0.2475921439420274, 0.006088649842999206, 0.31203183112392907, -0.036869203979483754, -0.07475746464056504, -0.029297797064479717  …  0.16867368684091563, -0.09450564983271922, -0.0587273302122711, -0.1326667940553803, -0.25530237980444614, 0.37556905374043376, 0.04922612067677609, 0.2605362549983866, -0.21871556587505667, -0.22915883767386164], [0.03295085436260177, -0.971861604433394, 0.034748713521512035, -0.0494065013245799, -0.01767479281403355, 0.0465459739459587, 0.007470494722096038, 0.003227960072276129, 0.0058328596338402365, -0.037591237446692356  …  0.03205152122876297, 0.11331109854742015, 0.03044900529526686, 0.017971704993311105, -0.009329252062960229, -0.02939354719650879, 0.022088835776251863, -0.02546111553658854, -0.0026257225461427582, 0.005702111697172774], [0.06968243992532257, -0.7119502191435176, -0.18136614593117445, -0.1695926215673451, 0.01725015359973796, -0.00694164951158388, -0.34621134287344574, 0.024709256792651912, -0.1632255805999673, -0.2158226433583082  …  -0.14153772108081458, -0.11256850346909901, 0.045109821764180706, -0.1162754336222613, -0.13221711766357983, 0.005365354776191061, 0.012750671705879105, -0.018208207549835407, 0.12458753932455452, -0.31843587960340897], [-0.19830349374441875, -0.6086693423968884, 0.08552341811170468, 0.35781519334042255, 0.15790663648524367, 0.02712571268324985, 0.09855601327331667, -0.05840653973421127, -0.09546429767790429, -0.13414717696055448  …  -0.0430935804718714, 0.2678584478951765, 0.08780994289014614, 0.01613469379498457, 0.0516187906322884, -0.07383067566731401, -0.1481272738354552, -0.010532317187265649, 0.06555344745952187, -0.1506167863762911], [-0.04347524125197773, -0.6327981074196994, -0.221116680035191, 0.0282207467940456, -0.0855024881522933, 0.12821801740178346, 0.1779499563280024, -0.10247384887512365, 0.0396432464100116, -0.0582580338112627  …  0.1253893207083573, 0.09628202269764763, 0.3165295473947355, -0.14915034201394833, -0.1376727867817772, -0.004153096613530293, 0.09277957650773738, 0.05917264554031624, -0.12230262590034507, -0.19655728521529914], [-0.10173946348675116, -0.6475660153977272, 0.1260284619729566, -0.11933160462857616, -0.04774310633937567, 0.09093928358804217, 0.041662676324043114, -0.1264739543938265, 0.09605293126911392, -0.16790474428001648  …  -0.04056684573478108, 0.09351665120940456, 0.15259195558799882, 0.0009949298312580497, 0.09461980828206303, 0.3067004514287283, 0.16129258773733715, -0.18893664085007542, -0.1806865244492513, 0.029319680436405825], [-0.251780954320053, -0.39147463259941456, -0.24359579328578626, 0.30179309757665723, 0.21658893985206484, 0.12304585275893232, 0.28281133086451704, 0.029187615341955325, 0.03616243507191924, 0.029375588909979152  …  -0.08071746662465404, -0.2176101928258658, 0.20944684921170825, 0.043033273425352715, -0.040505542460853576, 0.17935596149079197, -0.08454569418519972, 0.0545941597033932, 0.12471741052450099, -0.24314124407858329], [0.28156471341150974, -0.6708572780452595, -0.1410302363738465, -0.08322589397277698, -0.022772599832907418, -0.04447265789199677, -0.016448068022011157, -0.07490911512503738, 0.2778432295769144, -0.10191899088372378  …  -0.057272155080983836, 0.12817478092201395, 0.04623814480781884, -0.12184190164369117, 0.1987855635987229, -0.14533603246124993, -0.16334072868597016, -0.052369977381939437, 0.014904286931394959, -0.2440882678882144], [0.12108727495744157, -0.714787344982596, 0.01632521838262752, 0.04437570556908449, -0.041199280304144284, 0.052984488452616, 0.03796520200156107, 0.2791785910964288, 0.11530429924056099, 0.12178223160398421  …  -0.07621847481721669, 0.18353870423743013, -0.19066653731436745, -0.09423224997242206, 0.14596847781388494, -0.09747986927777111, 0.16041150122587072, -0.02296513951256738, 0.06786878373578588, 0.15296635978447756]], 0)</pre>


<div class="markdown"><p>To not get too many entries let's just look at the first 20 iterations again</p></div>

<pre class='language-julia'><code class='language-julia'>R4 = gradient_descent(
    M,
    F3,
    gradF,
    data[1];
    record=[:Count =&gt; RecordCount()],
    return_state=true,

)</code></pre>
<pre class="code-output documenter-example-output" id="var-R4">RecordSolverState{DebugSolverState{GradientDescentState{Vector{Float64}, Vector{Float64}, StopWhenAny{Tuple{StopAfterIteration, StopWhenGradientNormLess}}, ArmijoLinesearch{ExponentialRetraction, Manopt.var"#58#61"}, ExponentialRetraction}}, NamedTuple{(:Iteration,), Tuple{RecordGroup}}}(DebugSolverState{GradientDescentState{Vector{Float64}, Vector{Float64}, StopWhenAny{Tuple{StopAfterIteration, StopWhenGradientNormLess}}, ArmijoLinesearch{ExponentialRetraction, Manopt.var"#58#61"}, ExponentialRetraction}}(GradientDescentState{Vector{Float64}, Vector{Float64}, StopWhenAny{Tuple{StopAfterIteration, StopWhenGradientNormLess}}, ArmijoLinesearch{ExponentialRetraction, Manopt.var"#58#61"}, ExponentialRetraction}([0.003348564083448399, -0.9989177042553917, 0.01260395651475293, 3.724195907654294e-5, 0.00347688650874772, 0.007393312784223247, 0.0015131386301483914, -0.020957965200961437, 0.014862725158453362, -0.007213435087886661  …  -0.003341405141763289, 0.004841935454254808, -0.010592700175885091, -0.013017126493473067, 0.0033263346448692836, -0.004530004405886269, 0.0030077996849159536, -0.015610320483091028, -0.0016794415453862106, -0.002537572063969514], [-8.697873855618293e-12, -2.3160719388626728e-12, -1.2354431122539047e-11, 8.988731274876729e-11, -4.733831441395371e-11, -1.5449887164975876e-11, 8.260573948232642e-11, 9.896137755831605e-11, 1.5104898581919102e-10, -1.0275020011262306e-11  …  1.1865317292602574e-10, -9.099753063698465e-11, -1.1517346820179297e-10, 2.3269229384591424e-11, 7.209477754913964e-11, 5.79698539876234e-11, 6.894389193660283e-12, 5.861714630665208e-11, -1.2133124491003552e-10, -1.6233332061389716e-11], IdentityUpdateRule(), ArmijoLinesearch{ExponentialRetraction, Manopt.var"#58#61"}(1.0, ExponentialRetraction(), 0.95, 0.1, 1.758086915910625, 0.0, Manopt.var"#58#61"()), StopWhenAny{Tuple{StopAfterIteration, StopWhenGradientNormLess}}((StopAfterIteration(200, ""), StopWhenGradientNormLess(1.0e-9, "The algorithm reached approximately critical point after 60 iterations; the gradient norm (3.865928812772072e-10) is less than 1.0e-9.\n")), "The algorithm reached approximately critical point after 60 iterations; the gradient norm (3.865928812772072e-10) is less than 1.0e-9.\n"), ExponentialRetraction()), Dict{Symbol, Manopt.DebugAction}()), (Iteration = RecordGroup(Manopt.RecordAction[RecordCount([25, 29, 33, 37, 40, 44, 48, 52, 56, 60  …  224, 228, 232, 236, 238, 243, 247, 255, 258, 261])], Dict(:Count =&gt; 1)),))</pre>

<pre class='language-julia'><code class='language-julia'>get_record(R4)</code></pre>
<pre class="code-output documenter-example-output" id="var-hash163156">60-element Vector{Tuple{Int64}}:
 (25,)
 (29,)
 (33,)
 (37,)
 (40,)
 (44,)
 (48,)
 ⋮
 (238,)
 (243,)
 (247,)
 (255,)
 (258,)
 (261,)</pre>


<div class="markdown"><p>We can see that the number of cost function calls varies, depending on how many linesearch backtrack steps were required to obtain a good stepsize.</p></div>

<!-- PlutoStaticHTML.End -->
```
