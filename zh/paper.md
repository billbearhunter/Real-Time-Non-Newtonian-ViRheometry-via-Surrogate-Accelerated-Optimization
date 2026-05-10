# 《Fast Non-Newtonian ViRheometry via Mixture-of-GP Surrogates》中文对照版

> 英文标题:*Fast Non-Newtonian ViRheometry via Mixture-of-GP Surrogates*
>
> **本文为英文 `body-journals.tex` + `acmtog-SIGGRAPH-submission.tex` 的一对一中文对照版本**(非投稿版本,仅用于阅读)。每段、每条公式、每条引用与英文版严格对齐;数学符号、引用 key、章节 / 表 / 图 / 公式标签均与英文版完全一致。

---

## 摘要

我们估计复杂流体——糊、含块状物的酱料、颗粒浆——的三个 Herschel--Bulkley 参数(屈服应力 $\sigma_Y$、幂律指数 $n$、稠度参数 $\eta$);常规流变仪无法对这类流体给出有用的测量。我们的做法是把溃坝视频镜头与基于物理的仿真匹配,沿用~\cite{hamamichi2023nonnewtonian} 的协议。该反演精确但缓慢:每次估计要求内循环数百次基于物质点法的评估,因而每个 setup 在 GPU 上耗时约八小时,所以该方法只是一种研究工具而不是常规测量。

我们将这一反演从大约八小时压缩到每个 setup 不到一分钟,做法是用一组拟合于 MPM 溃坝仿真语料上的高斯过程专家混合体替换内循环仿真器。我们进一步用一条基于 ChArUco 标志物与剪影边缘精化的算法化两阶段管线,替换反演之前的手动相机标定流程。

我们的验证方法是:在真实视频采集上对比模拟在环反演前作,在流变仪可以处理的材料上对比平板流变仪测量,并发现恢复出来的参数与前作持平,而延迟低三个数量级。秒级反演解锁三种在每个 setup 数小时代价下不可达的用途:\emph{(i)} 实时跟踪材料参数的演化;\emph{(ii)} 在单次测量会话内表征多种材料、批次或条件;\emph{(iii)} 迭代测量工作流——每次采集恢复出来的参数指导下一次调整。

---

## 1. Introduction

许多日常流体——酸奶、糊、酱料、浆、化妆品乳液——表现出非牛顿行为,这种行为决定了它们如何倾倒、铺展、附着、形变。要定量表征这种行为,需要测量随剪切率变化的应力,传统上使用平板或锥板流变仪。但是,这些仪器在一类实用上重要的材料上失效:含有大于仪器测量间隙的内含物的材料——含块状物的酱料、颗粒悬浮液、含固形物的食物——以及流变性质演化速度快于多分钟实验流程能够捕捉的材料。

非牛顿视频流变~\cite{hamamichi2023nonnewtonian} 弥补这一空缺:它从一次简单溃坝流的观测中估计 Herschel--Bulkley 参数 $(\sigma_Y, \eta, n)$,而无需任何仪器接触材料。其底层原理是反演仿真:在一个优化器内部反复执行物质点法 (MPM) 前向模型,直到仿真出来的流再现观测剪影。由于前向模型基于物理,这种方法天然能处理仪器流变仪无法处理的那些材料,并且只用一段短视频就能恢复完整的本构三元组。

然而,这个本来独具一格的方法被它内循环的成本所卡住。一次 MPM 溃坝仿真在现代 GPU 硬件上要花数分钟,而无导数优化通常要在收敛前发出数百次候选评估。因此端到端反演\emph{每个 setup 大约八小时},使该方法被困在离线研究用途,排除了视频方法本该可以达到的三种能力:

- \emph{跟踪时变流体的参数。}当材料的性质在测量窗口内演化——沉降、老化、温度漂移、混合历史、烹饪冷却——一个小时尺度的反演返回的是描述过去某一状态的参数。等到结果给出来,所推断的值已经过期。
- \emph{在单次会话中表征多种材料。}比较若干品牌或批次,或者把单一材料在温度、稀释或剪切历史条件下做扫描,需要把反演运行 $N$ 次。在每个 setup 数小时的代价下,即使温和的 $N$ 也不切实际,所以每材料流变只能局限在精挑细选的展示样本上,无法扩展到调查或目录。
- \emph{迭代测量工作流。}闭环用法——每次采集恢复出来的参数指导下一次对配方或工艺的调整——在测量到决策的延迟为小时级时被堵死。

我们提出一个代理加速框架,把反演从每个 setup 数小时压缩到几秒,使三种能力在一个实用工作流内同时可用。我们的技术核心耦合三个组件。

\emph{(i)}~一个\emph{高斯过程专家分层混合}在优化器内部替换 MPM。生产代理 bank 按容器几何、终端流距、以及归一化轨迹形状对前向映射做分区。运行时,可观测三元组 $(W,H,\mathbf{y})$ 把每次查询确定性地路由到一个局部精确 GP 专家,最后用一次 source-aware 观测证据重排,使被选专家与观测来源和 runout regime 对齐。

\emph{(ii)}~一个\emph{GP-aware 反演似然}处理观测到的剪切率范围以稠度项为主时出现的屈服应力 / 稠度脊。CMA-ES 最小化一个 log 空间的 Type-II 边缘似然目标,该目标使用被路由 GP 的预测方差、一个观测噪声地板,以及对应的 Occam 项。残余的脊通过多起点散度和局部 Laplace 置信区间来报告,而不是被一个人工调出来的屈服应力先验掩盖。

\emph{(iii)}~一条\emph{算法化相机标定管线}替换前作中位于反演之前的 fSpy + Photoshop 手工流程。一块 ChArUco 标志物板提供初始位姿估计,后续的剪影边缘精化把相机对齐到溃坝几何上,无需逐次采集的人工标注,移除了采集流程中最大的固定成本。

我们的工作继承自~\cite{hamamichi2023nonnewtonian} 的问题设定、溃坝观测协议和反演形式,并贡献了使该形式在计算上变得切实可行的机器。\cite{hamamichi2023nonnewtonian} 用一条多阶段协议处理屈服应力 / 稠度退化,该协议结合了基于损失 Hessian 的相似性分析、对 Hessian 的平面 Poiseuille 流代理、Hessian 引导的后续 setup 主动选择、以及迭代多 setup CMA-ES;协议本身是健全的,在 $30$ 个材料-试验案例上得到了验证,但它继承了每次迭代的 MPM 成本。通过把反演管线移植到一个学习好的代理之上,同样的 setup 选择与两 setup 联合机器变成每材料可承担,而不只是每展示项可承担,而反演返回时间在上述三种用途所需的延迟之内。主要贡献是:

- 一个\emph{高斯过程专家分层混合},它在反演内循环里用一组冻结的、被路由的精确 GP 专家替换 MPM;这些专家按几何、轨迹长度和归一化形状结构化,以处理 Herschel--Bulkley 前向映射的非平稳性。
- 一个\emph{GP-aware 反演},它直接在似然中使用预测不确定度,在被路由的子问题内对 CMA-ES 做温启动并加边界,并通过重启和基于 Hessian 的不确定度诊断报告残余的屈服应力 / 稠度脊。
- 一条\emph{算法化两阶段相机标定管线},基于 ChArUco 标志物和剪影边缘精化,替换前作的手工流程,移除每次采集的最大固定成本。
- 一项评估,确立秒级视频流变开启了在每个 setup 数小时代价下不可达的三种用途:\emph{(i)}~实时跟踪材料参数的演化;\emph{(ii)}~在单次测量会话内表征多种材料、批次或条件;\emph{(iii)}~迭代测量工作流——每次采集恢复出来的参数指导下一次调整。

---

## 2. Related Work

### 2.1 Physical Measurements and Flow Rheology Signals

定量流变学传统上用流变仪(平板、锥板、转筒)测量,这类仪器在已知剪切率下施加受控层流,以恢复剪切应力与剪切率之间的本构关系。这些仪器要求一个窄间隙来强制清晰的剪切廓形,因而排除了带毫米级内含物的材料,以及流变性质演化速度快于实验流程能够捕捉的材料。控制更弱但更易用的替代方案(如 Bostwick consistometer)用一次简单的溃坝试验给出单个可读测量值。

重力驱动流动作为一种与理想流动流变仪互补的流变学观测有着长久的脉络。\citet{pashias1996fifty} 的坍落度试验从一次几何读数恢复屈服应力;\citet{roussel2005fiftycent} 把这一论证推广到与溃坝几何直接对应的铺展段;停止后的自由表面流动的最终沉积形状本身就是一种流变学指纹~\cite{coussot1996rheological};\citet{ancey2007plasticity} 把更广义的重力驱动屈服应力流定位为一种标准流变测量情境。流前沿位置与 Herschel--Bulkley 参数之间的关系由\citet{liumei1989slow} 和~\citet{balmforth2007two} 的相似性分析定量化,这也是~\citet{hamamichi2023nonnewtonian} 所借用的同一套解析框架。

我们的前向仿真器继承了把连续介质仿真带入图形学的物质点法 (MPM) 脉络~\cite{stomakhin2013material,jiang2015affine,hu2018moving,jiang2016material},剪切稀化和屈服应力流变性通过 return-mapping 形式实现~\cite{yue2015continuum,ram2015material,klar2016drucker,tampubolon2017multi,wolper2019cdmpm}。我们在基于 Taichi 的实现~\cite{hu2019taichi}上采用~\citet{yue2015continuum} 的 Herschel--Bulkley return map;来自我们直接前作的同一实验室的、面向酱料的剪切稀化仿真器~\cite{nagasawa2019mixing} 与我们关心的材料类别一致。面向图形学的流体仿真的更广义介绍见~\cite{bridson2015fluid}。


### 2.2 Video-Based Material Estimation

通过把渲染动力学与观测动力学匹配来从视频估计仿真参数,在图形学与视觉中有长久的脉络~\cite{bhat2002computing,monszpart2016smash}。Takahashi 与 Lin~\cite{takahashi2019video} 通过逐场景匹配渲染剪影与观测溃坝剪影确立了"视频到参数"的范式(针对粘性流体);Hamamichi~et~al.~\cite{hamamichi2023nonnewtonian} 把这一范式扩展到非牛顿 Herschel--Bulkley 三元组 $(\sigma_Y,\eta,n)$,是我们工作的直接前作。他们处理屈服应力/稠度退化的方式综合了:基于损失 Hessian 的相似性分析、对 Hessian 的平面 Poiseuille 流代理、Hessian 引导的后续 setup 主动选择、以及迭代多 setup CMA-ES;协议本身是健全的,在 $30$ 个材料-试验案例上得到了验证,但它继承了每次迭代的 MPM 成本。我们继承他们的溃坝观测协议、他们的 Herschel--Bulkley 反演目标、以及他们的模拟在环 formulation,贡献的是代理机器,使同一套多 setup 机器对每材料都付得起,而非只能 showcase。

相关工作还从其他视觉模态恢复材料参数:基于视频的布料摩擦~\cite{rasheed2020visual}、通过物理引导网络的粒子图像测速流变~\cite{thakur2025learning}、以及标量粘度的消费视频神经回归~\cite{rauzan2024protorheology}。这些目标针对不同的材料类或观测模态,并不恢复完整的 Herschel--Bulkley 三元组。从实验视频提取自由表面在~\cite{cochard2009tracking} 中讨论。反向流变在地球物理流场景中也曾被用浅水前向模型与伴随方法处理过~\cite{martin2010inverse};我们的形式化采用的是完整的三维 MPM 前向模型,以及在一个学习好的代理之上做无导数反演,因此不同。

一条平行路径通过可微仿真追求参数恢复。可微 MLS-MPM 家族~\cite{hu2019chainqueen,hu2020difftaichi,huang2021plasticinelab} 已被应用到形变与塑性材料;gradSim~\cite{murthy2021gradsim} 演示了端到端的可微物理与可微渲染结合。与从视频做非牛顿恢复最相关的两条路径中,PAC-NeRF~\cite{li2023pacnerf} 把可微 MPM 与 NeRF 渲染耦合,从多视角视频恢复粘度参数;Eastman~\cite{eastman2024recovery} 通过可微 MPM 与神经渲染从一段番茄酱柱坍塌视频恢复 Herschel--Bulkley 参数。这些路径逐场景优化,不摊销跨材料可复用的代理。PlasticineLab~\cite{huang2021plasticinelab} 的证据表明穿过非光滑塑性 return map 的梯度是病态的,这一点促使我们在一个概率代理之上选择无梯度反演。


### 2.3 Identifiability of Free-Surface Observations

把模拟在环反演替换成代理反演的一个前提条件是,所选择的观测 $\mathbf{y}_{\mathrm{obs}}$ 要承载足以识别 $(\sigma_Y,\eta,n)$ 的信息。近期的反问题分析工作确立了仅靠自由表面演化就足够:\citet{martinpritchard2022inferring} 证明完整的广义牛顿本构函数原则上可以由自由表面演化唯一恢复;\citet{ahmadi2022identification} 计算了三个 Herschel--Bulkley 参数关于自由表面读数的线性独立灵敏度系数;\citet{muchiri2024identification} 演示了在粘塑性流变学下从浅水高程水文图反演屈服应力和稠度参数。我们的标量流前沿轨迹 $\mathbf{y}_{\mathrm{obs}}$ 是这种演化的低维总结,因此在~\S\ref{sec:inverse} 中处理的残余屈服应力 / 稠度退化\emph{不是}手机视频观测的某种基本信息论极限,而是在低 $\sigma_Y$ 区(屈服项被粘性贡献压倒)出现的实际病态条件。


### 2.4 Surrogate-Accelerated Inverse

高斯过程回归~\cite{rasmussen2006gaussian} 是昂贵模拟器的天然代理,因为它把灵活的非参数回归与校准的预测不确定度耦合在一起。当数据规模超出精确 GP 的可承受范围时,稀疏与变分近似~\cite{snelson2006sparse,quinonero2005unifying,titsias2009variational,hensman2013gaussian,matthews2016sparse} 用一个诱导变量近似替换完整协方差。单个全局 GP 不足以拟合具有非均质局部结构的函数;专家混合 (MoE) 模型~\cite{jordan1994hierarchical,yuksel2012twenty} 把预测通过门控机制分解到多个专家上,GP 专家混合用 GP 专家实例化这一思路~\cite{tresp2001mixtures,rasmussen2002infinite,meeds2006alternative,nguyen2008local,yuan2008variational,trapp2020deep}。聚合独立 GP 预测有专门方法:Bayesian Committee Machine~\cite{tresp2000bayesian} 及其变体~\cite{cao2014generalized,deisenroth2015distributed,liu2018generalized} 提供闭式的方差加权融合;稀疏专家加 committee 聚合最近也被应用到物理仿真模拟~\cite{cohen2024sparse}。

反演昂贵模拟器在文献里归属\emph{计算机模型的贝叶斯校准}~\cite{kennedy2001bayesian} 和\emph{基于仿真的推断}~\cite{cranmer2020frontier} 两条标题之下,在目标昂贵且无导数的情形下,主流搜索策略是 efficient global optimization~\cite{jones1998efficient} 和贝叶斯优化~\cite{snoek2012practical,shahriari2016taking}。"在 GP 代理上做无导数优化"这一模式~\cite{bajaj2019gaussian,hansen2016cma} 已被应用到反向粘塑性~\cite{vandamme2021bayesian} 和粘弹性~\cite{hashemi2025bayesian}。在无导数反演内部用一个训练好的代理替换昂贵模拟器并不是启发式做法:\citet{stuart2018posterior} 证明 GP 代理后验与真实模拟器后验之间的 Hellinger 距离被代理预测误差的矩界住;\citet{teckentrup2020convergence} 把这一保证推广到核超参由数据估计的现实情形。因此把 MPM-in-the-loop 替换成一个学好的 GP 代理继承的是同一条贝叶斯校准~\cite{kennedy2001bayesian} 与现代基于仿真的推断~\cite{cranmer2020frontier} 的后验保真度合法性。


### 2.5 Camera Calibration

从图像估计相机位姿与内参是计算机视觉的基础任务;多视几何的教科书处理见~\cite{hartley2004multiple}。基于标志物的标定通过检测到的 fiducial 给出闭式初始位姿:AprilTag~\cite{olson2011apriltag}、ARTag~\cite{fiala2005artag}、以及 ArUco / ChArUco 系列~\cite{garrido2014automatic,garrido2016generation} 都给出 3D-2D 对应关系,可由 PnP 变种(如 EPnP~\cite{lepetit2009epnp})求解。\citet{forstner1987fast} 的基于梯度的角点检测器做亚像素角点精化,以在位姿恢复之前锐化对应关系。除了基于标志物的初始化以外,基于剪影或边缘的精化通过在欧氏距离变换~\cite{borgefors1986distance,maurer2003linear}上的 chamfer-distance 优化把相机对齐到物体边界,边缘图来自 Canny 检测器~\cite{canny1986computational}。我们的标定流水线(\S\ref{sec:calib})把 ChArUco 初始位姿与两阶段剪影边缘精化结合起来,替代~\cite{hamamichi2023nonnewtonian} 所用的手动 fSpy + Photoshop 流程,移除每次采集的最大固定成本。


## 3. Background

### 3.1 Herschel--Bulkley Rheology

许多非牛顿流体可由 Herschel--Bulkley (HB) 模型~\cite{herschel1926konsistenzmessungen}良好近似,该模型把剪切应力响应表示为
$$
    \tau \;=\; \sigma_Y \;+\; \eta\,\dot\gamma^{\,n},
    \tag{eq:hb-scalar}
$$
其中 $\tau$ 是剪切应力,$\dot\gamma$ 是剪切率,$\sigma_Y$ 是屈服应力,$\eta$ 是稠度参数,$n \in (0, 1]$ 控制剪切稀化 ($n < 1$) 与牛顿行为 ($n = 1$)。在三维下,偏应力张量为
$$
    \boldsymbol{\tau}^{\mathrm{dev}}
    \;=\;
    \left[\;\frac{\sigma_Y}{\|\mathbf{D}\|}
    \;+\; \eta\,\|\mathbf{D}\|^{\,n-1}\;\right] \mathbf{D},
    \qquad
    \|\mathbf{D}\|
    = \sqrt{\tfrac{1}{2}\mathbf{D}\!:\!\mathbf{D}},
    \tag{eq:hb-tensor}
$$
其中 $\mathbf{D}$ 是应变率张量。可允许应力区域由 von Mises 屈服函数定界
$$
    \Phi_{Y}(\boldsymbol{\tau})
    \;=\; \|\boldsymbol{\tau}^{\mathrm{dev}}\| \;-\; \sigma_Y \;\le\; 0,
    \tag{eq:hb-yield}
$$
塑性流动律~(\ref{eq:hb-tensor}) 仅在 $\Phi_{Y} > 0$ 时被激活。我们采用 Yue 等人的弹塑性 MPM 实现~\cite{yue2015continuum},该实现通过一个在每个塑性步上强制 $\Phi_{Y} \le 0$ 的 Newton--Raphson return map 在稳态恢复同一个 $(\sigma_Y, \eta, n)$ 三元组。HB 响应因此由三参数向量
$$
    \boldsymbol{\theta} \;=\; (\sigma_Y,\,\eta,\,n) \in \Theta \subset
    \mathbb{R}^{3}
$$
刻画。这一参数三元组与既有非牛顿视频流变~\cite{hamamichi2023nonnewtonian} 所用的相同;我们把本构模型当作给定,聚焦在以视频帧率延迟恢复 $\boldsymbol{\theta}$ 上。


### 3.2 Dam-Break MPM Forward Model

按照~\cite{hamamichi2023nonnewtonian} 的实验协议,我们通过观测一段从宽 $W$、高 $H$ 的矩形液柱在平底无滑移地板上释放出来的溃坝流来获得 $\boldsymbol{\theta}$。前向仿真器把~\S3.1 的 HB return map 集成在一个网格间距 $\Delta x \approx 10^{-1}$\,cm、时步 $\Delta t = 5\times 10^{-4}$\,s 的 moving-least-squares MPM~\cite{hu2018moving} 网格上,基于 Taichi~\cite{hu2019taichi} 实现。观测是一个流前沿位移向量,在释放后八个均匀间隔的帧 $t_1,\dots,t_8$ 上采样:
$$
    \mathbf{y}
    \;=\;
    \bigl(x_1,\,x_2,\,\dots,\,x_8\bigr)^{\!\top}
    \;\in\;
    \mathbb{R}_{\ge 0}^{8},
    \tag{eq:y-def}
$$
其中 $x_i$ 是流体在时刻 $t_i$ 的最大横向铺展(以 cm 为单位)。由构造 $x_1 \le x_2 \le \dots \le x_8$,因为流前沿不会回缩。我们把前向仿真器记作 $\mathbf{y} = \mathcal{S}(\boldsymbol{\theta};\,W, H)$。

\paragraph{实现细节。}
前向仿真器有三个细节值得提到。\emph{(i)~容器壁}使用 sticky 边界:每步把内部网格单元上的动量清零,任何穿入壁的粒子被标记为 interior 并从下游剪影中排除,使得嵌入壁内的流体不会在 $\mathcal{S}(\boldsymbol{\theta}; W, H)$ 中产生伪亮像素。\emph{(ii)~流体表面}通过对栅格化带符号距离场做 Laplacian-then-biharmonic 平滑提取,每一平滑步把单元上的 $\varphi$ clamp 到由局部粒子半径界限所允许的范围 $[\varphi_{\min},\,\varphi_{\max}]$ 内;这种自动 clamp 移除了早期管线中的逐帧 $\varphi$ 调整。\emph{(iii)~渲染剪影}由零等值面的 marching-cubes 网格化产生,然后通过~\S\ref{sec:calib} 的标定相机 $(\mathbf{K}_{0}, R^{*}, \mathbf{t}^{*})$ 投影,从 MPM 粒子闭合到 $8$ 维流距向量 $\mathbf{y}$ 的回路。


### 3.3 Observation Pipeline

从一段手机视频计算观测向量 $\mathbf{y}_{\mathrm{obs}}$ 需要两步预处理:(a)~相对于实验场景恢复相机位姿;(b)~在定义~(\ref{eq:y-def}) 中 $\mathbf{y}$ 的八个均匀间隔帧上提取基于剪影的流前沿距离。剪影提取步骤遵循~\cite{hamamichi2023nonnewtonian} 的协议(背景去除、二值分割、前沿铺展测量)。相机标定步骤是我们的贡献,作为方法的一部分详细见~\S\ref{sec:calib}。


### 3.4 Inverse Video Rheometry as Simulation-in-the-Loop Optimization

给定一次在几何 $(W, H)$ 下从真实实验采集到的观测 $\mathbf{y}_{\mathrm{obs}}$,视频流变通过把仿真器输出与观测匹配来恢复 $\boldsymbol{\theta}$,
$$
    \hat{\boldsymbol{\theta}}
    \;=\;
    \arg\min_{\boldsymbol{\theta}\in\Theta}\;
    \mathcal{L}\bigl(\boldsymbol{\theta};\,
    \mathbf{y}_{\mathrm{obs}},\,W,\,H\bigr),
    \tag{eq:inverse-generic}
$$
其中
$\mathcal{L}(\boldsymbol{\theta}) = \|\mathcal{S}(\boldsymbol{\theta};\,W,H)
- \mathbf{y}_{\mathrm{obs}}\|^{2}$ 是观测匹配损失。由于 $\mathcal{S}$ 由一次 MPM 积分定义,$\mathcal{L}$ 在每次询问下都很贵——在现代 GPU 硬件上每次几分钟——并且穿过塑性 return map 是非光滑的。\cite{hamamichi2023nonnewtonian} 的模拟在环 formulation 在一个无导数优化器内部直接求 $\mathcal{S}$,在每一个候选 $\boldsymbol{\theta}$ 上付一次完整 MPM 仿真的代价。

损失景观带有一个众所周知的识别性病态。定义把屈服主导 regime 与粘性主导 regime 分隔开的\emph{特征剪切率}
$$
    \dot\gamma^{\star}(\sigma_Y, \eta, n)
    \;=\; \bigl(\sigma_Y / \eta\bigr)^{1/n}.
    \tag{eq:gammastar}
$$
把~(\ref{eq:hb-scalar}) 改写为
$$
    \tau(\dot\gamma)
    \;=\; \eta\,\dot\gamma^{\,n}\left[\,1
    \;+\; \left(\frac{\dot\gamma^{\star}}{\dot\gamma}\right)^{\!n}\,\right],
    \tag{eq:hb-factored}
$$
表明当 $\dot\gamma / \dot\gamma^{\star} \to \infty$ 时,第二个方括号趋向 $1$,应力就渐近地由 $(\eta, n)$ 单独支配;形式上,
$$
    \left.
      \frac{\partial \tau}{\partial \sigma_Y}
    \right/\!
    \left.
      \frac{\partial \tau}{\partial \eta}
    \right|_{\dot\gamma \gg \dot\gamma^{\star}}
    \;=\;
    \frac{1}{\dot\gamma^{\,n}}
    \;\longrightarrow\; 0.
    \tag{eq:ridge}
$$
如果观测到的溃坝 regime 大部分位于 $\dot\gamma^{\star}$ 之上,$\mathbf{y}_{\mathrm{obs}}$ 就实际上对 $\sigma_Y$ 不敏感,而仍然约束 $(\eta, n)$,因此存在一个连续的近等损失解集合,以 $\sigma_Y$ 与一个补偿性增加的 $\eta$ 来交换。\citet{hamamichi2023nonnewtonian} 通过量纲相似性分析识别出这一退化,并提出多 setup 联合反演作为一种缓解方法,以 $N$ 个 setup 估计的代价付 $N$ 倍的模拟在环代价。

此外,当目标材料本身在测量窗口内演化——沉降、老化、温度漂移、混合、烹饪冷却——真实参数状态 $\boldsymbol{\theta}_{t}$ 在反演期间漂移,而一个小时尺度的估计就不再描述当前状态。我们因此希望估计延迟短到足以\emph{跟踪}演化中的状态,而不只是降低摊销之后的计算成本。


## 4. Method (Surrogate-Accelerated Inverse)
\label{sec:method-block}

### 4.1 Problem Statement
\label{sec:problem}

我们保留~(\ref{eq:inverse-generic}) 的物理前向模型,但把每次内循环的 MPM 调用替换成一个冻结的专家混合代理。在线估计器接收一次标定后的溃坝观测 $\mathbf{y}_{\mathrm{obs}}\in\mathbb{R}^{8}$ 与 setup 几何 $(W,H)$,把观测路由到一小组局部 GP 专家上,并用一个 GP-aware Type-II 极大似然损失求解 $\boldsymbol{\theta}=(n,\eta,\sigma_Y)$。同样的机器通过让两次同材料的观测共享一个 $\boldsymbol{\theta}$ 来处理两 setup 情形。

图~\ref{fig:overview} 总结了实现的管线。离线时,MPM 仿真按几何与观测到的流前沿轨迹分区,每个分区训练一个精确 GP 专家。在线时,可观测三元组 $(W,H,\mathbf{y}_{\mathrm{obs}})$ 通过一个确定性三阶段 gate 选出一个主子专家,再由一次 source-aware 观测证据重排做精细化;CMA-ES 然后在被路由专家的填充训练 box 内做优化。一次 setup 之后还可以通过~\citet{hamamichi2023nonnewtonian} 的解析 Hessian-正交规则建议下一次物理实验,我们原样采用其闭式 Fisher 信息构造。

\begin{figure*}[t]
    \centering
    \includegraphics[width=\textwidth]{figs/method_overview.pdf}
    \caption{随附代码实现的生产管线。用户提供一个观测到的流前沿向量 $\mathbf{y}_{\mathrm{obs}}$ 与 setup 几何 $(W,H)$。一组冻结的局部精确 GP 专家先把观测通过几何与 y-shape 分区路由,然后在一个有界 CMA-ES 反演内部计算一个 GP-aware 似然。两 setup 材料情形下,两次观测共享一个 $\boldsymbol{\theta}$ 的同时各自保留被路由专家。}
    \Description{生产反演管线的方框图:观测提取、分层路由到一个局部 GP 专家、GP-aware 似然、CMA-ES 反演,以及可选的两 setup 联合恢复。}
    \label{fig:overview}
\end{figure*}

\paragraph{形式化陈述。}\label{sec:problem-formal}
原始的模拟在环反演是
$$
    \hat{\boldsymbol{\theta}}
    =
    \arg\min_{\boldsymbol{\theta}\in\Theta}
    \left\|\mathcal{S}(\boldsymbol{\theta};W,H)
    -\mathbf{y}_{\mathrm{obs}}\right\|^2 .
    \tag{P}
$$
我们的单 setup 生产估计器把 $\mathcal{S}$ 替换为一个被路由的 GP 代理 $\tilde{\mathcal{S}}_{k^\star}$,并最小化负对数边缘似然
$$
    \hat{\boldsymbol{\theta}}
    =
    \arg\min_{\mathbf{z}\in\Theta_{k^\star}}
    \mathcal{L}_{\mathrm{GP}}\!\left(
      \tilde{\mathcal{S}}_{k^\star}(\mathbf{z};W,H),
      \mathbf{y}_{\mathrm{obs}}\right)
    + p_{\mathrm{sup}}\,
      \mathcal{L}_{\mathrm{box}}(\mathbf{z};k^\star),
    \tag{P$'$}
$$
其中 $\mathbf{z}=(n,\log\eta,\log\sigma_Y)$,$k^\star$ 由 $(W,H,\mathbf{y}_{\mathrm{obs}})$ 选出,$\Theta_{k^\star}$ 是被路由的子专家的填充训练 box。对同一材料的两个 setup $a, b$,条件独立性给出联合目标
$$
    \hat{\boldsymbol{\theta}}
    =
    \arg\min_{\mathbf{z}\in\Theta_{a,b}}
    \mathcal{L}_{\mathrm{GP}}^{(a)}(\mathbf{z})
    + \mathcal{L}_{\mathrm{GP}}^{(b)}(\mathbf{z})
    + p_{\mathrm{sup}}\,
      \tfrac{1}{2}\!\left(
      \mathcal{L}_{\mathrm{box}}^{(a)}(\mathbf{z})+
      \mathcal{L}_{\mathrm{box}}^{(b)}(\mathbf{z})\right).
    \tag{P$^{\star}$}
$$
当前的生产路径不使用显式的 $\sigma_Y$ 锚定先验;沿 HB ridge 的不确定度通过多起点散度和一个局部 Laplace Hessian 报告。

\begin{table}[!htbp]
\centering
\footnotesize
\setlength{\tabcolsep}{4pt}
\caption{相对于规范模拟在环反演,本文实现的替代方案。}
\label{tab:subs}
\begin{tabular}{@{}p{0.30\linewidth}p{0.48\linewidth}p{0.14\linewidth}@{}}
\toprule
难点 & 生产替代方案 & 章节 \\
\midrule
前向代价
  & $\mathcal{S}\mapsto\tilde{\mathcal{S}}_{k^\star}$:被路由的局部精确 GP 专家
  & \S\ref{sec:surrogate} \\
非平稳性
  & 观测条件路由与逐子专家填充 box
  & \S\ref{sec:surrogate}, \S\ref{sec:inverse} \\
尺度与模型不确定度
  & 使用 GP 预测方差与一个观测噪声地板的 log 空间 Type-II ML 损失
  & \S\ref{sec:inv-loss} \\
Ridge 诊断
  & 多重启 CMA-ES 散度与 Laplace 95\% CI
  & \S\ref{sec:inv-cma} \\
\bottomrule
\end{tabular}
\end{table}

### 4.2 Input and Output Representation
\label{sec:repr}

每次代理查询使用五维物理输入
$$
    \mathbf{x}=(n,\eta,\sigma_Y,W,H)^\top ,
$$
对 $\eta$ 与 $\sigma_Y$ 在标准化前取对数,
$$
    \tilde{\mathbf{x}}
    =
    \frac{\mathbf{x}_{\log}-\boldsymbol{\mu}_X}
         {\boldsymbol{\sigma}_X},
    \qquad
    \mathbf{x}_{\log}
    =
    (n,\log\eta,\log\sigma_Y,W,H)^\top .
    \tag{eq:xscale}
$$
输出是八帧流前沿向量 $\mathbf{y}\in\mathbb{R}^{8}$;输出尺度变换减去一个冻结的训练均值,
$$
    \tilde{\mathbf{y}}=\mathbf{y}-\boldsymbol{\mu}_Y .
    \tag{eq:yscale}
$$
反演时,观测到的向量 $\mathbf{y}_{\mathrm{obs}}$ 既作为似然数据也作为路由信号。

### 4.3 Hierarchical Mixture-of-GP Surrogate
\label{sec:surrogate}

\paragraph{三层确定性分区。}
\label{sec:surr-3layer}
前向映射被分成 $G$ 个几何条件子 bank,每个子 bank 装一组局部拟合的 GP 专家。一次查询通过下式被路由
$$
    (W,H,\mathbf{y}_{\mathrm{obs}})
    \xrightarrow{L_1} g^\star
    \xrightarrow{L_2} b^\star
    \xrightarrow{L_3} s^\star .
    \tag{eq:gate}
$$
Layer 1 把 setup 几何分配到 $G$ 个几何索引之一,
$$
    g^\star = G_\Pi(W,H), \qquad g^\star\in\{1,\ldots,G\},
    \tag{eq:layer1}
$$
其中 $G_\Pi$ 是定义在标定的 $(W,H)$ 训练网格上的固定最近-cell 映射。在被选中的几何子 bank 内,Layer 2 把终端流位移 $y_8$ 放进存好的长度 bin
$$
    b^\star
    =
    \mathrm{bin}\!\left(y_{\mathrm{obs},8};
    \{e_j^{(g^\star)}\}\right),
    \tag{eq:layer2}
$$
对位于经验范围之外的样本 clamp 到最近 bin。Layer 3 使用归一化的流形状特征
$$
    \boldsymbol{\phi}(\mathbf{y})
    =
    \frac{(y_2-y_1,\ldots,y_8-y_1)^\top}{\max(y_8,\epsilon)}
    \tag{eq:phi}
$$
通过 bin-specific scaler 标准化、由 bin-specific PCA 投影,并分配到最近的 K-means 质心,
$$
    s^\star
    =
    \arg\min_s
    \left\|
      \mathrm{PCA}_{g^\star,b^\star}(\boldsymbol{\phi})
      -\mathbf{c}_{g^\star,b^\star,s}
    \right\|_2 .
    \tag{eq:layer3}
$$
最后一阶段是观测证据重排。在 PCA-最近候选子专家集合 $\{s\}$ 中,我们按每个候选的训练输出云对实测 $\mathbf{y}_{\mathrm{obs}}$ 的解释程度打分,然后保留一个主子专家。部署时根据观测来源选择两个重排 profile 之一。

\paragraph{\texttt{real\_robust}(相机部署)。}
真实视频带有不可忽略的提取噪声和模型差异,尤其在早期帧上更明显。因此 robust profile 只使用终端位移 $y_{\mathrm{obs},8}$,也就是观测轨迹中最稳定的标量摘要。把子专家 $s$ 内训练 $y_8$ 值近似为高斯分布,$y_{\mathrm{obs},8}$ 在子专家 $s$ 下的负对数边缘似然就退化为一维标准化距离
$$
    s_{\mathrm{real}}
    =
    \arg\min_s
    \frac{|y_{\mathrm{obs},8}-\mathrm{median}(Y_{s,8})|}
         {\max(\mathrm{std}(Y_{s,8}),\,0.1)} ,
    \tag{eq:y8q}
$$
这是基于边缘证据的贝叶斯模型选择的一维标准情形~\cite{murphy2012machine,mackay2003information}。

\paragraph{\texttt{sim\_precision}(合成验证)。}
当 $\mathbf{y}_{\mathrm{obs}}$ 是由与代理训练语料相同的 MPM 仿真器生成时,八个轨迹分量都是有效的 simulator evidence。因此 simulator-matched profile 选择其训练输出云中包含与 $\mathbf{y}_{\mathrm{obs}}$ 最近的 frame-weighted 轨迹的子专家,
$$
    s_{\mathrm{sim}}
    =
    \arg\min_s\;
    \min_{i\in s}
    \sqrt{
      \frac{1}{8}
      \sum_{d=1}^{8}
      \alpha_d^2
      \bigl(Y_{s,i,d}-y_{\mathrm{obs},d}\bigr)^2
    },
    \tag{eq:simprecision}
$$
其中 $\{Y_{s,i,\cdot}\}_i$ 是分配给子专家 $s$ 的轨迹,$\alpha_d$ 是反演损失使用的 frame weights(\S\ref{sec:inv-nn};表~\ref{tab:hyper})。两个 profile 都返回单一主子专家 $s_{\mathrm{prod}}\in\{s_{\mathrm{real}},s_{\mathrm{sim}}\}$,反演直接使用这个 top-1 专家,不在候选之间做集成聚合。

\paragraph{为何使用确定性路由。}
\label{sec:surr-deterministic}
反演时材料参数未知,但 $(W,H,\mathbf{y}_{\mathrm{obs}})$ 是被观测到的。在可观测量上做路由避开了一个会要求未知 $\boldsymbol{\theta}$ 的循环 gate,并通过检查所选子专家的训练 box 使越界情形可被诊断。

\paragraph{逐 cell 高斯过程专家。}
\label{sec:surr-experts}
每个子专家是八个独立精确 GP 回归器的元组,共享同一个标度后的 $5$-D 输入,并使用 Mat\'ern-$5/2$ ARD 核
$$
    k_{5/2}(\mathbf{a},\mathbf{b})
    =
    \sigma_f^2
    \left(1+\sqrt{5}r+\tfrac{5}{3}r^2\right)
    e^{-\sqrt{5}r},
    \quad
    r^2=\sum_{d=1}^{5}\frac{(a_d-b_d)^2}{\ell_d^2}.
    \tag{eq:matern}
$$
对输出帧 $d$,后验均值与方差为
$$
\begin{aligned}
    \mu_{s,d}(\tilde{\mathbf{x}})
    &=
    \mathbf{k}_*^\top
    (K_s+\sigma_{n,s,d}^2 I)^{-1}\tilde{\mathbf{y}}_{s,d}, \\
    \sigma_{s,d}^2(\tilde{\mathbf{x}})
    &=
    k_{5/2}(\tilde{\mathbf{x}},\tilde{\mathbf{x}})
    -
    \mathbf{k}_*^\top
    (K_s+\sigma_{n,s,d}^2 I)^{-1}\mathbf{k}_* .
\end{aligned}
\tag{eq:gp-predictive}
$$
超参数通过最大化精确 GP 边缘似然来拟合,
$$
\begin{aligned}
    \log p(\tilde{\mathbf{y}}_{s,d}\mid X_s)
    =
    &-\tfrac{1}{2}
      \tilde{\mathbf{y}}_{s,d}^{\top}
      (K_s+\sigma_{n,s,d}^2I)^{-1}
      \tilde{\mathbf{y}}_{s,d}
      -\tfrac{1}{2}\log|K_s+\sigma_{n,s,d}^2I| \\
    &-\tfrac{|X_s|}{2}\log 2\pi .
\end{aligned}
\tag{eq:gp-mll}
$$

\paragraph{Top-1 预测。}
\label{sec:surr-top1}
预测器直接使用被路由的主子专家
$$
    \tilde{\mathcal{S}}(\boldsymbol{\theta};W,H)
    =
    \mu_{s_{\mathrm{prod}}}(\tilde{\mathbf{x}}).
    \tag{eq:top1}
$$
由于~(\ref{eq:gate})--(\ref{eq:y8q}) 的确定性 gate 已经选出一个局部支撑充分的单一专家,无需做专家间聚合:在路由之上做 top-$K$ 集成只会引入 gate 已经排除掉的区域的方差。

\paragraph{单调后处理。}
\label{sec:surr-mono}
物理上的流前沿距离不减,因此被解码出的 GP 均值通过下式做单调化
$$
    \hat{y}_d\leftarrow \max_{j\le d}\hat{y}_j,
    \qquad d=1,\ldots,8 .
    \tag{eq:mono}
$$

\begin{algorithm}[!htbp]
\caption{离线构建轨迹形状 GP bank。}
\label{alg:train}
\KwData{MPM 语料 $\{(\boldsymbol{\theta}_i,W_i,H_i,\mathbf{y}_i)\}$。}
\KwOut{由 $25$ 个几何条件子 bank 组成的冻结 bank,每个子 bank 持有标度参数、路由元数据、子专家分配,以及逐子专家的精确 GP 回归器。}
\BlankLine
拟合全局输入 / 输出标度参数(式~(\ref{eq:xscale})、(\ref{eq:yscale}))\;
\ForEach{几何子 bank $g$}{
  从 $y_8$ 构造长度 bin,并在每个 bin 内通过~(\ref{eq:phi}) 构造 PCA / K-means 形状 cluster\;
  合并过小的 cluster,并冻结分配表\;
  \ForEach{子专家 $s$}{
    按~(\ref{eq:gp-mll}) 训练八个独立的精确 GP 回归器(每帧一个)\;
    存储训练输入 / 输出、验证误差,以及边缘似然重排所用的逐子专家汇总统计 $\mathrm{median}(Y_{s,8})$、$\mathrm{std}(Y_{s,8})$\;
  }
}
\Return{生产模型 bank。}
\end{algorithm}

\begin{algorithm}[!htbp]
\caption{在线前向 $\hat{\mathbf{y}}=\tilde{\mathcal{S}}(\boldsymbol{\theta};W,H)$。}
\label{alg:forward}
\KwIn{$\boldsymbol{\theta}$、几何 $(W,H)$、观测路由向量 $\mathbf{y}_{\mathrm{obs}}$。}
\KwOut{单调的预测流前沿向量 $\hat{\mathbf{y}}$。}
\BlankLine
通过~(\ref{eq:gate})--(\ref{eq:y8q}) 把 $(W,H,\mathbf{y}_{\mathrm{obs}})$ 路由出 $s_{\mathrm{prod}}$\;
标准化 $\mathbf{x}_{\log}$ 并求被选 ExactGP 的值(式~(\ref{eq:gp-predictive}))\;
把均值解码到物理单位并应用~(\ref{eq:mono})\;
\Return{$\hat{\mathbf{y}}$ 与 GP 预测 std。}
\end{algorithm}

### 4.4 GP-Aware Inverse
\label{sec:inverse}

\paragraph{子专家内温启动。}
\label{sec:inv-nn}
CMA-ES 在被路由子专家的最近训练点处初始化(以观测空间度量),
$$
    \mathbf{z}^{\mathrm{NN}}
    =
    \mathbf{z}_{i^\star},\quad
    i^\star
    =
    \arg\min_{i\in\mathcal{D}_{s_{\mathrm{prod}}}}
    \frac{1}{8}\sum_{d=1}^{8}
    \alpha_d^2\left(y_{i,d}-y_{\mathrm{obs},d}\right)^2 .
    \tag{eq:nn}
$$
帧权重 $\boldsymbol{\alpha}=(\alpha_1,\ldots,\alpha_8)$ 形成一个非递减时间表:由构造,溃坝序列的早期帧含有 gate 释放瞬态(有限时间 gate 抽离、释放定时抖动、自由表面与表面张力等 MPM 前向未建模的瞬态效应),而晚期帧已经稳定到代理可信的稳态铺展 regime。给晚期帧加权使残差与前向模型可信的 regime 对齐。我们实验中使用的具体时间表是一个冻结超参数,列在~\S\ref{sec:data}(表~\ref{tab:hyper})。

\paragraph{逐子专家边界收紧。}
\label{sec:inv-box}
搜索 box 是被路由子专家的支撑 $[\mathbf{z}_{\min},\mathbf{z}_{\max}]$,在 $z$ 空间内以其标准差 $\mathbf{s}$ 的某一比例 $p$ 填充,并 clip 到全局 HB 边界,
$$
    \Theta_{k^\star}
    =
    \Theta\cap
    [\mathbf{z}_{\min}-p\mathbf{s},\,
     \mathbf{z}_{\max}+p\mathbf{s}].
    \tag{eq:tight-box}
$$
目标在未填充的支撑之外加一个权重为 $p_{\mathrm{sup}}$ 的软二次罚项,使优化器在子专家训练数据靠近某面稀疏时不被强制严格遵守边界。$p$ 与 $p_{\mathrm{sup}}$ 的数值列在~\S\ref{sec:data}。

\paragraph{无显式 $\sigma_Y$ 先验下的 Ridge 诊断。}
\label{sec:inv-sy}
Herschel--Bulkley 观测可能在一个 $(\eta,\sigma_Y)$ ridge 上仍然弱可识别。我们不使用人工选择的 $\sigma_Y$ 先验来打破这条 ridge;残余的不确定度通过 CMA-ES 最优解的多重启散度,以及一个局部 Laplace--Hessian 置信区间(\S\ref{sec:inv-cma})来暴露,因此报告的不确定度被绑定在似然与数据上,而不是绑定在用户挑选的锚定上。

\paragraph{GP-aware Type-II ML 损失。}
\label{sec:inv-loss}
对于一个候选 $\mathbf{z}$,被选中的 GP 同时给出 $\hat{y}_d(\mathbf{z})$ 与预测标准差 $\sigma_{\mathrm{GP},d}(\mathbf{z})$。我们在 log 空间工作,把通过 delta 法传播的 GP 认知方差与一个吸收残余真实-仿真差异的同方差观测噪声地板 $\sigma_{\mathrm{obs,log}}^2$ 卷积起来:
$$
    v_d(\mathbf{z})
    =
    \left(
    \frac{\sigma_{\mathrm{GP},d}(\mathbf{z})}
         {\max(\hat{y}_d(\mathbf{z}),\epsilon)}
    \right)^2
    + \sigma_{\mathrm{obs,log}}^2 .
    \tag{eq:gp-var}
$$
逐 setup 的负对数边缘似然为
$$
    \mathcal{L}_{\mathrm{GP}}(\mathbf{z})
    =
    \frac{1}{8}\sum_{d=1}^{8}
    \left[
      \frac{\alpha_d^2\, r_d(\mathbf{z})^2}{2v_d(\mathbf{z})}
      +\frac{1}{2}\log v_d(\mathbf{z})
    \right],
    \qquad
    r_d=\log\hat{y}_d-\log y_{\mathrm{obs},d}.
    \tag{eq:obs-loss}
$$
log-方差项是 Occam 因子:它阻止优化器仅因为某些区域 GP 预测方差大就偏好那些区域。噪声地板 $\sigma_{\mathrm{obs,log}}$ 在所有报告的实验中保持固定而不是逐材料调,其具体数值见~\S\ref{sec:data}。

\paragraph{优化器与不确定度输出。}
\label{sec:inv-cma}
我们用 CMA-ES~\cite{hansen2016cma} 优化~(\ref{eq:Pprime}),一种把多元高斯采样分布自适应到损失景观上的无导数演化策略。CMA-ES 很适合~(\ref{eq:obs-loss}) 产生的分段光滑、ridge 易出现的曲面;它的自适应去掉了对临时步长时间表的需求。每次独立重启都用确定性种子;最低损失重启被报告为 MAP 估计,而重启最优解的标准差提供一个 truth-blind 的 ridge-退化诊断。在 MAP 处对~(\ref{eq:obs-loss}) 做有限差分 Hessian 给出 $(n,\eta,\sigma_Y)$ 的局部 Laplace 95\% 区间。我们实验中使用的种群大小、步长、代数预算和重启次数列在~\S\ref{sec:data}。

\paragraph{每代批量评估。}
\label{sec:inv-batch}
每一代 CMA-ES 在 $\lambda$ 个候选 $\mathbf{z}$ 上估值损失。由于精确 GP 原生支持批量测试输入((\ref{eq:gp-predictive}) 应用于一个堆叠的 $\mathbf{X}_*$)~\cite{rasmussen2006gaussian},整代由一次前向调用估值,而不是 $\lambda$ 次顺序调用;逐候选的 Mahalanobis、Occam 与支撑项随后在 GP 之外做聚合。这把 GPU 派发的固定逐次调用开销摊销到整代之上,且不改变优化轨迹;经验墙上时钟收益见~\S\ref{sec:results}。

\paragraph{联合两 setup 反演。}
\label{sec:inv-joint}
对同一材料的两次观测 $a, b$,两个被路由的 setup 各自保留自己的 GP 专家,但共享同一个 $\boldsymbol{\theta}$。生产实现最小化~(\ref{eq:Pstar});若两个子专家 box 重叠则使用其交集,否则它们的填充并集提供一个保守的搜索 box。setup~$a$ 的估计被作为联合阶段的温启动,使 setup~$b$ 在单 setup 后验之\emph{上}贡献信息,而不是重新启动搜索。

\begin{algorithm}[!htbp]
\caption{在线单 setup 反演。}
\label{alg:invert}
\KwIn{$\mathbf{y}_{\mathrm{obs}}$、$(W,H)$、生产 GP bank。}
\KwOut{$\hat{\boldsymbol{\theta}}$、后验诊断。}
\BlankLine
按~(\ref{eq:y8q}) 把 $(W,H,\mathbf{y}_{\mathrm{obs}})$ 路由到主子专家\;
按~(\ref{eq:tight-box})、(\ref{eq:nn}) 构造填充 box $\Theta_{k^\star}$ 与最近邻温启动\;
对~(\ref{eq:Pprime}) 跑多重启 CMA-ES\;
可选:在最优点求 Hessian 区间\;
\Return{$\hat{\boldsymbol{\theta}}$ 与诊断。}
\end{algorithm}

### 4.5 Hessian-Orthogonal Setup-2 Proposal (Inherited)
\label{sec:proposer}

第二个容器几何 $(W_2, H_2)$ 的选择不属于我们的方法贡献。我们原样使用~\citet{hamamichi2023nonnewtonian}(DOI \texttt{10.1145/3618310})的闭式、基于 Fisher 信息的提议器,在此仅做总结,以便后面的联合算法自洽。

给定从第一个几何 $(W_1, H_1)$ 得到的单 setup 估计 $\hat{\boldsymbol{\theta}}_1$,\citet{hamamichi2023nonnewtonian} 在 Herschel--Bulkley 本构律下用平面 Poiseuille 流近似溃坝,并推导出他们的相似性损失关于 $\boldsymbol{\theta}$ 的解析 Hessian $\mathbf{H}(\hat{\boldsymbol{\theta}}_1; W,H)$。设 $\mathbf{q}_1$ 是 $\mathbf{H}(\hat{\boldsymbol{\theta}}_1; W_1, H_1)$ 最小特征值对应的单位特征向量:这是 $\boldsymbol{\theta}$ 空间里第一个 setup 携带最少信息的方向。一个对联合估计有信息量的第二 setup,正当其主最少信息方向 $\mathbf{q}_2(W,H)$ 与 $\mathbf{q}_1$ 正交时,因为此时联合 Fisher 信息在 $(\mathbf{q}_1, \mathbf{q}_2)$ 子空间里满秩。提议器因此选取
$$
    (W_2^\star, H_2^\star)
    \;=\;
    \arg\min_{(W,H)\in\mathcal{W}}
    \bigl\langle \mathbf{q}_1,\,\mathbf{q}_2(W,H)\bigr\rangle^2 ,
    \tag{eq:proposer}
$$
其中 $\mathcal{W}$ 是标定的几何 box。由于 Hessian 是闭式的,求~(\ref{eq:proposer}) 不需要任何前向仿真或代理调用。

我们采用这一提议器而不做修改:我们的贡献是把消耗其输出的联合估计变成每材料而不是每 showcase 可承担。其实现是对~\citet{hamamichi2023nonnewtonian} 的公式忠实移植,任何与材料无关的推导步骤——相似性损失的定义、平面 Poiseuille 简化、以及解析 Hessian——都恰当归属给那篇论文。

\begin{algorithm}[!htbp]
\caption{在线联合两 setup 反演。}
\label{alg:joint}
\KwIn{同一材料的两次观测,$(\mathbf{y}^{(a)},W_a,H_a)$ 与 $(\mathbf{y}^{(b)},W_b,H_b)$。}
\KwOut{共享的 $\hat{\boldsymbol{\theta}}$。}
\BlankLine
独立把两 setup 各自路由到其主子专家\;
由两个子专家支撑构造联合搜索 box\;
对~(\ref{eq:Pstar}) 用多重启 CMA-ES 最小化求和后的 GP-aware 似然\;
\Return{共享的 $\hat{\boldsymbol{\theta}}$ 与两 setup 诊断。}
\end{algorithm}


## 5. Camera Calibration
\label{sec:calib}

为了在真实实验上跑反演,我们需要把溃坝流的手机视频转换成~(\ref{eq:y-def}) 中定义的观测向量 $\mathbf{y}_{\mathrm{obs}} = (x_{1}, \dots, x_{8}) \in \mathbb{R}_{\ge 0}^{8}$。我们使用一个两阶段相机标定再加基于剪影的流前沿提取;整条管线总结在算法~\ref{alg:calib} 中。

\paragraph{Stage 1:ChArUco 先验。}
一块印好的 ChArUco 板~\cite{garrido2014automatic,garrido2016generation} 被放在实验场景中,与流体容器并排。标准针孔投影~\cite{hartley2004multiple} 把世界点 $\mathbf{P}_{w}$ 映射到像素:
$$
    \mathbf{P}_{c} \;=\; \mathbf{R}\,\mathbf{P}_{w} + \mathbf{t},
    \qquad
    \mathbf{p} \;=\; \tfrac{1}{Z_{c}}\,\mathbf{K}\,\mathbf{P}_{c},
    \quad
    \mathbf{K} = \begin{pmatrix}
          f_{x} & 0     & c_{x} \\
          0     & f_{y} & c_{y} \\
          0     & 0     & 1
        \end{pmatrix}.
    \tag{eq:pinhole}
$$
我们不把 $(K, R, \mathbf{t})$ 当作 $11$ 个自由标量来优化,而是利用我们的设备水平、相机看着固定靶标这一事实,把整个相机用一个紧致的 $6$ 维向量参数化
$$
    \boldsymbol{\theta}_{\mathrm{cam}}
    \;=\;
    (l_{x},\,l_{z},\,\theta_{E},\,\phi_{E},\,
    \mathrm{fov},\,d_{\mathrm{eye}}),
    \tag{eq:theta-cam}
$$
其中 $\mathbf{l} = (l_{x}, 0, l_{z})^{\!\top}$ 是设备地面上的看向点,$(\theta_{E}, \phi_{E})$ 是眼方向的极角和方位角,$d_{\mathrm{eye}}$ 是眼到目标的距离,$\mathrm{fov}$ 是垂直视场。相机滚转通过构造固定(设备水平,渲染器强制 $\mathbf{C}_{x}$ 轴水平,见~(\ref{eq:basis}))。眼位置与一个正交相机基随后从 $\boldsymbol{\theta}_{\mathrm{cam}}$ 确定性地重建出来:
$$
    \mathbf{E}(\boldsymbol{\theta}_{\mathrm{cam}})
    \;=\; d_{\mathrm{eye}}
    \begin{pmatrix}
        \sin\theta_{E}\cos\phi_{E} \\
        \cos\theta_{E} \\
        -\sin\theta_{E}\sin\phi_{E}
    \end{pmatrix},
    \tag{eq:eye}
$$
$$
    \mathbf{C}_{z} = \frac{\mathbf{E} - \mathbf{l}}
                           {\|\mathbf{E} - \mathbf{l}\|},
    \quad
    \mathbf{C}_{x} = \frac{(C_{z,3},\,0,\,-C_{z,1})^{\!\top}}
                           {\|(C_{z,3},\,0,\,-C_{z,1})\|},
    \quad
    \mathbf{C}_{y} = \mathbf{C}_{z} \times \mathbf{C}_{x},
    \tag{eq:basis}
$$
所以 $R = [\mathbf{C}_{x}\,|\,\mathbf{C}_{y}\,|\,\mathbf{C}_{z}]^{\!\top}$ 且 $\mathbf{t} = -R\,\mathbf{E}$;$\mathbf{C}_{x}$ 的构造强制相机的上轴水平。一个世界点 $\mathbf{P}_{i}$ 因此先映射到相机坐标
$$
    \mathbf{Y}_{i}
    \;=\;
    \bigl(
      (\mathbf{P}_{i} - \mathbf{E})\!\cdot\!\mathbf{C}_{x},\;
      (\mathbf{P}_{i} - \mathbf{E})\!\cdot\!\mathbf{C}_{y},\;
      (\mathbf{P}_{i} - \mathbf{E})\!\cdot\!\mathbf{C}_{z}
    \bigr),
    \tag{eq:screen}
$$
然后通过透视除法到像素。匹配渲染器的垂直视场约定 $f_{y} = H/(2\tan(\mathrm{fov}/2))$ 给出尺度因子
$$
    s \;=\; \frac{1}{2\,\tan(\mathrm{fov}/2)},
    \tag{eq:scale}
$$
$\mathbf{P}_{i}$ 到图像平面 $[0, W]\times[0, H]$ 的完整重投影读作
$$
    \pi(\mathbf{P}_{i};\,\boldsymbol{\theta}_{\mathrm{cam}})
    \;=\;
    \left(
        \frac{W}{2} + s\,H\,\frac{[\mathbf{Y}_{i}]_{1}}{|[\mathbf{Y}_{i}]_{3}|},\;
        \frac{H}{2} - s\,H\,\frac{[\mathbf{Y}_{i}]_{2}}{|[\mathbf{Y}_{i}]_{3}|}
    \right).
    \tag{eq:project}
$$

在检测到的 ChArUco 板上做亚像素角点精化~\cite{forstner1987fast} 之后,Stage~1 求解
$$
    \hat{\boldsymbol{\theta}}_{\mathrm{cam}}
    \;=\;
    \arg\min_{\boldsymbol{\theta}_{\mathrm{cam}}}\,
    \frac{1}{|\mathcal{I}|}
    \sum_{i \in \mathcal{I}}
    \rho_{\delta}\!\bigl(
      \|\mathbf{u}_{i} - \pi(\mathbf{P}_{i};\,\boldsymbol{\theta}_{\mathrm{cam}})\|
    \bigr),
    \tag{eq:stage1}
$$
其中 $\mathcal{I}$ 索引检测到的角点,带像素观测 $\mathbf{u}_{i}$ 与已知板坐标 $\mathbf{P}_{i}$,$\rho_{\delta}$ 是阈值 $\delta = 2$\,px 的 Huber 损失,
$$
    \rho_{\delta}(e)
    \;=\;
    \begin{cases}
        \tfrac{1}{2}e^{2}, & e < \delta, \\
        \delta\bigl(e - \tfrac{1}{2}\delta\bigr), & e \ge \delta,
    \end{cases}
    \tag{eq:huber}
$$
它阻止任何单个未精化好的角点主导拟合。该最小化使用 Nelder--Mead~\cite{nelder1965simplex},因为 $\pi$ 容易求值但在 $\boldsymbol{\theta}_{\mathrm{cam}}$ 参数化下难求导;\citet{gao2012implementing} 的自适应变种被保留给下面非光滑的 Phase~B,在那里其更大的收缩步长有助于逃离 IoU 目标的像素栅格平台。Stage~1 返回一个内参矩阵 $\mathbf{K}_{0}$(由 $\mathrm{fov}$ 与图像尺寸推出)和一个初始外参 $(R_{0}, \mathbf{t}_{0})$(由 $\mathbf{E}, \mathbf{l}$ 通过~(\ref{eq:basis}) 推出)。

\paragraph{Stage 2:剪影引导的外参精化。}
Stage~1 通常精确到几个像素,但可能留下小的残余偏移,因为 ChArUco 板与容器并不是刚性耦合的。Stage~2 \emph{固定} $\mathbf{K}_{0}$ 而仅精化外参,使渲染的容器剪影对齐到溃坝释放瞬间采集的参考帧 $I_{0}$。我们使用一个两阶段时间表。

\emph{Phase~A(光滑 chamfer)。}
对 $I_{0}$ 应用 Canny 边检测~\cite{canny1986computational} 得到一个目标边像素集 $E$,我们由此预先计算一个精确欧氏距离变换~\cite{maurer2003linear,borgefors1986distance}
$$
    D_{E}(p) \;=\;
    \min_{q \in E}\,\|\,p - q\,\|_{2}
    \qquad \text{对每个像素 } p \in \Omega.
    \tag{eq:edt}
$$
对候选外参 $(R, \mathbf{t})$,记 $P_{f}(R, \mathbf{t})$ 为第 $f$ 个可见立方体面上的投影边像素,$\mathcal{F}$ 为可见面集合。逐面归一化的加权 chamfer 代价为
$$
    \mathcal{L}_{\mathrm{cham}}(R, \mathbf{t})
    \;=\;
    \frac{1}{|\mathcal{F}|}
    \sum_{f \in \mathcal{F}}
    \frac{\sum_{p \in P_{f}(R,\mathbf{t})}\, w(p)\, D_{E}(p)}
         {\sum_{p \in P_{f}(R,\mathbf{t})}\, w(p)},
    \tag{eq:chamfer}
$$
其中 $D_{E}(p)$ 由 EDT 双线性内插得到,$w(p)$ 是三个侧向加权 $w_{\mathrm{left}}(p_{x})$、$w_{\mathrm{top}}(p_{y})$、$w_{\mathrm{bottom}}(p_{y})$ 的乘积。逐面归一化阻止长边(如底壁)主导短边。我们用 L-BFGS-B~\cite{byrd1995limited,zhu1997algorithm} 在 Rodrigues 旋转向量~\cite{rodrigues1840lois} 与平移的六个 DoF 上最小化 $\mathcal{L}_{\mathrm{cham}}$,box-clamp 在 Stage-$1$ 先验 $(\mathbf{r}_{0}, \mathbf{t}_{0})$ 周围 $\pm 2.5^{\circ}$ 与 $\pm 2$\,cm。

\emph{Phase~B(非光滑 IoU)。}
从 Phase-$A$ 解出发,我们最大化渲染剪影 $M(R, \mathbf{t})$ 与目标前景 $T$ 之间的加权 Jaccard 相似度 / 像素 IoU~\cite{jaccard1912distribution}
$$
    \mathrm{IoU}_{w}(R, \mathbf{t})
    \;=\;
    \frac{\sum_{p}\, w(p)\,[\,M(p)\wedge T(p)\,]}
         {\sum_{p}\, w(p)\,[\,M(p)\vee T(p)\,]},
    \tag{eq:iou}
$$
使用一个大小为 $0.15^{\circ}/1.5$\,mm 的自适应 Nelder--Mead~\cite{gao2012implementing} 单纯形。由于 $\mathrm{IoU}_{w}$ 在像素栅格上离散化,它是不可微的;光滑 chamfer 代价上的 L-BFGS-B 阶段提供了避开~(\ref{eq:iou}) 的组合局部极小值所需的温启动。

\paragraph{安全 clamp。}
精化后我们检查相对 Stage-$1$ 先验的偏离 $\Delta_{\mathrm{rot}} = \|\mathbf{r}^{*} - \mathbf{r}_{0}\|$ 与 $\Delta_{t} = \|\mathbf{t}^{*} - \mathbf{t}_{0}\|$;若任一超过安全阈值(默认 $3^{\circ}$ 或 $3$\,cm),回退到 ChArUco 外参。安全阈值设得比 Phase-$A$ box($2.5^{\circ}/2$\,cm)略松,使无界的 Phase-$B$ Nelder--Mead 原则上可以越过 box 但只接受温和的越界;这对两个阶段的强局部极小值都起到防护作用,尤其是在 ChArUco 板放置粗糙时。

\paragraph{流前沿提取。}
后续每一帧 $I_{1}, \dots, I_{8}$ 在 $t_{1}, \dots, t_{8}$ 采集,通过 HSV-亮度阈值化(默认阈值 $128$,黑色前景)接着 $3\times 3$ 核~\cite{gonzalez2018digital} 上的形态学开闭操作做二值化;只保留最大连通前景分量以剔除杂散斑点。然后我们提取帧 $i$ 上前景 mask 的外轮廓 $\partial\Omega_{i}$,把每一个轮廓像素 $\mathbf{u} = (u, v)$ 通过标定后的针孔相机反投影到世界地面 $y = 0$。设 $\mathbf{C}_{w} = -R^{\!\top}\,\mathbf{t}$ 为世界系相机中心,$\mathbf{d}(\mathbf{u}) = R^{\!\top}\,\mathbf{K}_{0}^{-1}\,(u, v, 1)^{\!\top}$ 为世界系射线方向,射线
$$
    \mathbf{P}_{w}(\tau)
    \;=\; \mathbf{C}_{w} + \tau\,\mathbf{d}(\mathbf{u})
    \tag{eq:backproj}
$$
在 $\tau^{\star} = -[\mathbf{C}_{w}]_{y}/[\mathbf{d}(\mathbf{u})]_{y}$ 处遇到地面。设 $\mathbf{v} \in \mathbb{R}^{2}$ 是世界 $xz$ 平面里的单位流向($\mathbf{v} = (1, 0)$ 对我们的装置成立),设 $x^{\mathrm{front}}_{\mathrm{box}}$ 为容器前壁位置,从实验清单中读出。帧 $t_{i}$ 的流距是 $\partial\Omega_{i}$ 沿 $\mathbf{v}$ 投影相对该前壁的最远有符号距离
$$
    x_{i} \;=\;
    \max_{\mathbf{u} \in \partial\Omega_{i}}
    \Bigl[\mathbf{v}^{\!\top}
          \bigl(\mathbf{P}_{w}(\tau^{\star}(\mathbf{u}))\bigr)_{xz}\Bigr]
    \;-\; x^{\mathrm{front}}_{\mathrm{box}}.
    \tag{eq:xi}
$$
最后,残余的二值化噪声偶尔会在帧之间引入向后跳跃;我们用一次单调 pass $x_{i} \leftarrow \max(x_{i}, x_{i-1})$(等价于在物理空间应用~(\ref{eq:mono}))来防护,产生算法~\ref{alg:invert} 与~\ref{alg:joint} 所消费的观测 $\mathbf{y}_{\mathrm{obs}} = (x_{1}, \dots, x_{8}) \in \mathbb{R}_{\ge 0}^{8}$。

\begin{algorithm}[!htbp]
\caption{Stage 1 重投影 oracle。}
\label{alg:reproj}
\KwIn{$6$-向量 $\boldsymbol{\theta}_{\mathrm{cam}}{=} (l_{x},l_{z},\theta_{E},\phi_{E},\mathrm{fov},d_{\mathrm{eye}})$;板坐标 $\{\mathbf{P}_{i}\}_{i\in\mathcal{I}}$;图像尺寸 $(W,H)$。}
\KwOut{投影像素 $\{\pi(\mathbf{P}_{i}; \boldsymbol{\theta}_{\mathrm{cam}})\}$。}
\BlankLine
设 $\mathbf{l}{=}(l_{x},0,l_{z})^{\!\top}$ 并按~(\ref{eq:eye}) 计算眼位置 $\mathbf{E}$\;
按~(\ref{eq:basis}) 构造正交基 $(\mathbf{C}_{x},\mathbf{C}_{y},\mathbf{C}_{z})$,使 $\mathbf{C}_{x}$ 水平;按~(\ref{eq:scale}) 取 $s{=}1/(2\tan(\mathrm{fov}/2))$\;
\ForEach{角点 $i\in\mathcal{I}$}{
  按~(\ref{eq:screen}) 求相机系坐标 $\mathbf{Y}_{i}$;按~(\ref{eq:project}) 投影到像素\;
}
\Return{$\{\mathbf{u}_{i}\}_{i\in\mathcal{I}}$。}
\end{algorithm}

\begin{algorithm}[!htbp]
\caption{两阶段相机标定。}
\label{alg:calib}
\KwIn{背景照片 $I_{\mathrm{bg}}$ 含 ChArUco 板;溃坝释放瞬间的参考帧 $I_{0}$;容器 $(W,H)$;图像尺寸 $(W,H)$;Phase-A box 边界 $(2.5^{\circ},2\,\mathrm{cm})$;安全阈值 $(3^{\circ},3\,\mathrm{cm})$;Huber $\delta{=}2$\,px。}
\KwOut{标定结果 $(\mathbf{K}_{0},R^{*},\mathbf{t}^{*})$。}
\BlankLine

\tcp*[h]{Stage 1:ChArUco 先验。}
在 $I_{\mathrm{bg}}$ 上检测 ChArUco 角点并做亚像素精化~\cite{garrido2014automatic,forstner1987fast}\;
对 $\boldsymbol{\theta}_{\mathrm{cam}}$ 跑 Nelder--Mead 最小化~(\ref{eq:stage1})、(\ref{eq:huber}) 的 Huber 重投影损失 $\mathcal{L}_{1}$,使用算法~\ref{alg:reproj} 求 $\pi$\;
按~(\ref{eq:eye})--(\ref{eq:scale}) 从 $\hat{\boldsymbol{\theta}}_{\mathrm{cam}}$ 提取 $(\mathbf{K}_{0},R_{0},\mathbf{t}_{0})$\;

\tcp*[h]{Stage 2:剪影引导的精化(固定 $\mathbf{K}_{0}$)。}
按~(\ref{eq:edt}) 计算 $I_{0}$ 上的 Canny 边集 $E$ 与 EDT $D_{E}$\;
\emph{Phase A:} 对~(\ref{eq:chamfer}) 的 chamfer 代价 $\mathcal{L}_{\mathrm{cham}}$ 在 $(\mathbf{r},\mathbf{t})$ 上跑 L-BFGS-B,box-clamp 在 $(\mathbf{r}_{0},\mathbf{t}_{0})$ 周围 $\pm\Delta_{\mathrm{rot}}^{\mathrm{box}}/\pm\Delta_{t}^{\mathrm{box}}$~\cite{byrd1995limited,zhu1997algorithm}\;
\emph{Phase B:} 从 Phase-A 解出发,对 $-\mathrm{IoU}_{w}$~(\ref{eq:iou}) 跑自适应 Nelder--Mead;单纯形半径 $(0.15^{\circ},1.5\,\mathrm{mm})$\;

\tcp*[h]{安全 clamp。}
\If{$\|\mathbf{r}^{*}{-}\mathbf{r}_{0}\|{>}\Delta_{\mathrm{rot}}^{\mathrm{safe}}$ \textnormal{或} $\|\mathbf{t}^{*}{-}\mathbf{t}_{0}\|{>}\Delta_{t}^{\mathrm{safe}}$}{
  回退 $(\mathbf{r}^{*},\mathbf{t}^{*}){\leftarrow}(\mathbf{r}_{0},\mathbf{t}_{0})$\;
}

\Return{$(\mathbf{K}_{0},R(\mathbf{r}^{*}),\mathbf{t}^{*})$。}
\end{algorithm}


## 6. Training Data and Frozen Configuration
\label{sec:data}

所有报告的生产结果使用同一个冻结的代理 bank,该 bank 训练一次,在每一个报告的实验中重用。该 bank 包含 $25$ 个几何条件子 bank,共持有 $1{,}396$ 个精确 GP 子专家。它们的分配表索引了 $425{,}831$ 条被路由的 MPM 训练行(每个几何子 bank 在 $13{,}581$ 与 $25{,}032$ 行之间),覆盖 $n \in [0.3,1.0]$、$\eta \in [10^{-3},3{\times}10^{2}]$、$\sigma_Y \in [10^{-3},4{\times}10^{2}]$、以及 $W,H\in[2,7]$ cm。我们不在该 bank 之上做任何材料相关的重训练或逐实验的超参数扫描。

\begin{table}[!htbp]
\centering
\small
\caption{贯穿整个评估所使用的冻结生产配置。}
\label{tab:hyper}
\begin{tabular}{@{}p{0.36\linewidth}p{0.58\linewidth}@{}}
\toprule
\textit{训练数据} & \\
\quad 分配行                        & $425{,}831$ 条被路由的 MPM 样本 \\
\quad 参数 box                      & $n\in[0.3,1.0]$,
                                      $\eta\in[10^{-3},300]$,
                                      $\sigma_Y\in[10^{-3},400]$ \\
\midrule
\textit{代理 bank} & \\
\quad 几何条件子 bank               & $25$ \\
\quad 子专家                        & 共 $1{,}396$;每个几何子 bank $24$--$71$ \\
\quad 路由                          & 几何 gate $+$ 长度 bin $+$ PCA / K-means 形状 $+$ source-aware 观测证据重排 \\
\midrule
\textit{逐 cell GP 专家} & \\
\quad 核                            & $\mathbb{R}^{5}$ 上的 Mat\'ern-$5/2$ ARD \\
\quad 输出                          & 每子专家 $8$ 个独立精确 GP \\
\midrule
\textit{路由} & \\
\quad profiles                      & \texttt{real\_robust}: terminal $y_8$ evidence;\quad \texttt{sim\_precision}: full-trajectory NN evidence \\
\quad 选中专家                      & 每查询 top-$1$;synthetic setup-2 diagnostic 中可保留 top-$3$ candidates \\
\midrule
\textit{反演 CMA-ES} & \\
\quad 种群 $P$                      & $12$ \\
\quad $\sigma_{0}$                  & $0.25$,在 $z=(n,\log\eta,\log\sigma_Y)$ 中 \\
\quad 代数 / 重启数                 & $30$ / $5$ \\
\quad box pad / 支撑权重            & $0.10$ / $0.25$ \\
\midrule
\textit{似然 / UQ} & \\
\quad 损失                          & log 空间的 GP-aware Type-II ML \\
\quad 噪声地板                      & $\sigma_{\mathrm{obs,log}}=0.20$ \\
\quad CI 报告                       & 数值 Hessian / Laplace 95\% CI \\
\bottomrule
\end{tabular}
\end{table}


## 7. Results
\label{sec:results}

### 7.1 Experimental Setup

我们针对四个问题评估 HVIMoGP-rBCM 与可识别性感知反演:(1)~代理多忠实地近似 MPM 前向模型,(2)~完整反演在留出样本上多准确地恢复 $\boldsymbol{\theta} = (\sigma_Y, \eta, n)$,(3)~代理相对于模拟在环反演给出多大的墙上时钟降幅,(4)~\S1 的三项贡献——freshness、流变仪不可行材料、多观测精度——在实验验证下是否成立。

\paragraph{数据集。}
我们使用三个评估集。\emph{$T_{\mathrm{sim}}$} 包含从 $295{,}419$ 行训练语料中按 $(n, \log\eta, \log\sigma_Y, W, H)$ 分层抽样留出的 $22{,}209$ 个 MPM 样本。\emph{$T_{\mathrm{inv}}$} 是一个合成基准,包含从训练 box 内抽取的 $30$ 种材料,每种在三个容器尺寸下渲染 MPM(共 $90$ 段视频)。\emph{$T_{\mathrm{real}}$} 由六到八种标准非牛顿流体(蜂蜜、番茄酱、蛋黄酱、洗发水、Carbopol 凝胶、以及一种乳液样品)组成,在我们的溃坝装置上用智能手机采集,并附带平板流变仪扫描。

\paragraph{基线。}
\emph{Baseline A} 是~\cite{hamamichi2023nonnewtonian} 的模拟在环管线:相同的观测、相同的 CMA-ES 超参数,但每个候选都由一次新的 MPM 运行求值,而不是由代理求值。Baseline~A 定义了模拟在环家族的精度上界与墙上时间下界。\emph{Baseline B} 是我们自己管线的一个简化版本,用来分离分层 gating 与可识别性感知反演的贡献:它在 $(W, H)$ 上使用 $K = 12$ 的扁平 Bayesian GMM、不带第二层 $\phi$ gate(共 $12$ 个专家)、top-$2$ 硬路由、以及一个不带温启动或 $\sigma_Y$ 先验的 CMA-ES 反演。

\paragraph{度量。}
对前向代理我们报告逐输出的平均绝对误差
$$
    \mathrm{MAE}(\hat{\mathbf{y}}, \mathbf{y})
    \;=\; \frac{1}{D_Y}\sum_{d=1}^{D_Y}|\hat y_d - y_d|,
    \tag{eq:mae}
$$
决定系数 $R^{2} = 1 - \sum_{i} \|\hat{\mathbf{y}}_i - \mathbf{y}_i\|^2 / \sum_i \|\mathbf{y}_i - \bar{\mathbf{y}}\|^2$,以及行最大误差分布($p_{50}, p_{95}, p_{99}, p_{999}$)。对反演我们报告逐参数相对误差
$$
    \mathrm{relE}(\hat\theta_d;\,\theta_d^{\mathrm{gt}})
    \;=\; \frac{|\hat\theta_d - \theta_d^{\mathrm{gt}}|}
               {|\theta_d^{\mathrm{gt}}|},
    \qquad d \in \{n, \eta, \sigma_Y\},
    \tag{eq:relE}
$$
在恢复参数处的前向 NMSE 作为对拟合的 sanity check,以及逐样本的墙上时钟。对照平板流变仪参考扫描的比较(\S\ref{sec:rheometer})我们进一步报告在实验有效剪切率区间 $\Gamma = [\dot\gamma_{\min}, \dot\gamma_{\max}]$ 上的积分 log-应力误差
$$
    \mathcal{E}(\hat{\boldsymbol{\theta}}, \boldsymbol{\theta}_{\mathrm{gt}})
    \;=\;
    \left(\frac{1}{|\Gamma|}
    \int_{\Gamma}
      \bigl[\log\hat\tau(\dot\gamma)
          - \log\tau_{\mathrm{gt}}(\dot\gamma)\bigr]^{2}
    \,d\log\dot\gamma\right)^{\!1/2}.
    \tag{eq:E}
$$
所有反演运行使用表~\ref{tab:hyper} 的冻结超参数与一个固定种子。


### 7.2 Surrogate Fidelity
\label{sec:fidelity}

我们首先验证训练好的代理是否足够准确地预测 MPM 输出,以正当化在反演内部替换仿真器。canonical test split($T_{\mathrm{sim}}$,$42{,}583$ 行)是训练语料严格留出的 $10\,\%$ slice,每一行都带有唯一子专家 assignment(训练时使用的 assignment,该子专家拟合时没有见过这一行)。对每个测试行,我们在该行的真值 $\boldsymbol{\theta}=(n,\eta,\sigma_Y)$ 与 setup geometry $(W,H)$ 处 forward assigned sub 的 exact GP,并把预测 $\hat{\mathbf{y}}$ 与 MPM 真值 $\mathbf{y}$ 对比。完整 $42{,}583$-row evaluation 访问了全部 $25$ 个几何子 bank 与全部 $1{,}491$ 个子专家。

\paragraph{Aggregate fidelity。}
跨八个 flow-front dimensions 与全部 $1{,}491$ 个子专家,代理达到 aggregate MAE $\mathrm{MAE}=0.038$\,cm 与 $R^2 = 0.9989$。median row-max absolute error 为 $0.029$\,cm,$p_{95}$ row-max 为 $0.26$\,cm,$p_{99}$ row-max 为 $0.58$\,cm,$p_{99.9}$ row-max 为 $1.92$\,cm。per-dim MAE 从第一帧的 $0.005$\,cm 温和上升到第八帧的 $0.064$\,cm(表~\ref{tab:fidelity}),反映了后期 flow-front distances 更大的动态范围;per-dim $R^2$ 在全部八帧上保持高于 $0.996$,其中 $d=8$ 的小幅下降($R^2 = 0.9967$)对应少数 subs 上的薄外推尾部。这些尾部行不会出现在 inverse loop 内部:GP-aware likelihood(\S\ref{sec:inv-loss}) 会 down-weight 超过 GP variance 的 residual,top-$k$ rerank(\S\ref{sec:surr-top1}) 也会把这类 query 从 boundary subs 路由开。

\begin{table}[!htbp]
\centering
\footnotesize
\setlength{\tabcolsep}{4pt}
\caption{$T_{\mathrm{sim}}$ 上的 surrogate forward fidelity($N=42{,}583$ held-out MPM rows,全部 $1{,}491$ 个子专家被访问)。逐维数字在所有行的 residual $\hat y_d - y_d$ 上计算。}
\label{tab:fidelity}
\begin{tabular}{@{}lccccccccc@{}}
\toprule
Frame $d$        & $1$ & $2$ & $3$ & $4$ & $5$ & $6$ & $7$ & $8$ & overall \\
\midrule
MAE (cm)         & 0.005 & 0.014 & 0.026 & 0.038 & 0.047 & 0.053 & 0.057 & 0.064 & 0.038 \\
$R^2$            & 0.9995 & 0.9996 & 0.9995 & 0.9994 & 0.9993 & 0.9992 & 0.9990 & 0.9967 & 0.9989 \\
\bottomrule
\end{tabular}
\end{table}

\paragraph{Per-sub regression quality。}
在至少收到五条 held-out MPM rows 的 $1{,}484$ 个子专家中,per-sub $R^2$ 的中位数为 $0.998$(图~\ref{fig:expert-r2})。lower $5\,\%$ tail 位于 $R^2 = 0.965$;这些是靠近 partition boundaries 或训练集最小的 subs(最小 sub 有 $63$ 条 MPM samples),但即便在这些 subs 上,held-out per-sub MAE 仍低于 $0.5$\,cm。upper $95\,\%$ tail 到达 $R^2 \approx 1.000$ 的浮点上限;这些是参数 box 稠密内部的 well-supported subs。单个最差子专家在 $25$ 条测试行上 $R^2 = 0.705$;它和接下来少数 low-$R^2$ subs 是最近 infill 的 local cells,训练集尚未 densify 到 production interior 的水平,而 inverse-time top-$1$ rerank(\S\ref{sec:surr-top1}) 除非 query 明确贴近边界,否则会避开这些 cells。

\begin{figure}[!htbp]
    \centering
    \includegraphics[width=\columnwidth]{figs/true_vs_pred.png}
    \caption{$N=42{,}583$ 个 held-out MPM rows 上的 surrogate parity,每个 flow-front frame $d=1\!\dots\!8$ 一个 panel。每行由其 training-time sub-expert assignment(oracle routing) forward;点是 $8{,}000$ 行随机 subsample。per-frame $R^2$ 与 MAE 在全部 $42{,}583$ 行上计算。点云在三个数量级的 flow-front distance 上贴近 identity line。}
    \Description{八个散点图,展示 42583 个 held-out samples 上 surrogate-predicted flow-front displacement 与 MPM ground truth 的关系;每个 frame 中点云都沿 identity line 紧密分布。}
    \label{fig:parity}
\end{figure}

\begin{figure}[!htbp]
    \centering
    \includegraphics[width=\columnwidth]{figs/moe_r2_boxplot.png}
    \caption{具有 $\ge 5$ 条 held-out test rows 的 $1{,}484$ 个子专家上的 per-sub-expert $R^2$ 分布,按 $25$ 个几何子 bank 分组。Median $R^2 = 0.998$(红虚线),$p_5 = 0.965$(灰点线),upper-tail subs 到达 $R^2 \approx 1.000$。最低 outliers 来自 $g\in\{1,2,5\}$ 中最近 infilled cells,其训练集尚未 densify 到 production interior 水平;inverse-time top-$1$ rerank 除非 query 明确贴近边界,否则会避开它们。}
    \Description{1484 个按 25 个几何子 bank 分组的子专家 R-squared 箱线图;分布集中在 0.99 以上,少数 outliers 来自 recently-infilled cells。}
    \label{fig:expert-r2}
\end{figure}

\paragraph{Posterior-uncertainty calibration。}
图~\ref{fig:calibration} 检查 routed GP posterior variance 是否可作为 inverse-time likelihood 的权重使用。我们把每个 frame 的 standardized residuals $r_d=(\hat y_d-y_d)/\sqrt{v_d}$ 按预测标准差 bin 分组;经验 RMSE 随 predicted $\sqrt{v_d}$ 单调上升,并落在同量级内。该校准不要求 GP posterior 完美 frequentist-calibrated;它只需要足够排序化,让 inverse loss 对高方差的 boundary predictions 施加较弱惩罚,对低方差的 interior predictions 施加较强惩罚。

\begin{figure}[!htbp]
    \centering
    \includegraphics[width=\columnwidth]{figs/sigma_calibration.png}
    \caption{GP posterior standard deviation 与 held-out absolute error 的校准曲线。每个点是一个 predicted uncertainty bin 上的 empirical absolute residual。单调趋势说明 posterior variance 可以在 Type-II inverse likelihood 中作为有用的 local reliability weight。}
    \Description{Predicted GP uncertainty 与 empirical held-out error 的校准图,显示 uncertainty 越大 error 越大。}
    \label{fig:calibration}
\end{figure}


### 7.3 Synthetic Validation at Scale
\label{sec:mainresults}

按照~\citet{hamamichi2023nonnewtonian} 的验证协议,我们首先在合成溃坝观测上测试反演——这些观测的真值参数是按构造已知的。我们对从~\S\ref{sec:data} 标定的参数 box 抽取的 $N$ 种合成材料运行完整管线(路由、GP-aware 似然、多重启 CMA-ES),并报告~\citet{hamamichi2023nonnewtonian} 引入的相对误差
$$
    E_{\mathrm{rel}}(\hat{\boldsymbol{\theta}},\boldsymbol{\theta}^{\!\star})
    \;=\;
    \sqrt{
      \biggl(\frac{\hat n - n^{\!\star}}{n_{\max}-n_{\min}}\biggr)^{\!2}
      +
      \biggl(\frac{\hat\eta - \eta^{\!\star}}{\eta_{\max}-\eta_{\min}}\biggr)^{\!2}
      +
      \biggl(\frac{\hat\sigma_Y - \sigma_Y^{\!\star}}{\sigma_{Y,\max}-\sigma_{Y,\min}}\biggr)^{\!2}
    }
    \tag{eq:erel}
$$
归一化由~\S\ref{sec:data} 的参数 box 给出。按他们的协议,当 $E_{\mathrm{rel}}\le 0.1$ 时我们宣布恢复成功。

本节所有合成面板都使用 Eq.~\eqref{eq:simprecision} 的 simulator-matched \texttt{sim\_precision} routing profile。论文后面的真实视频反演使用 Eq.~\eqref{eq:y8q} 的 \texttt{real\_robust} profile。两个 profile 共享同一套几何 bank、终端距离 bin、PCA/K-means 候选生成、top-$1$ 专家选择、GP 似然和 CMA-ES 反演;唯一切换的是最后的观测证据重排分数。因此合成实验是对代理反演的 simulator-matched precision test,而相机实验保留真实--仿真差异所需要的 noise-robust terminal-displacement evidence。

\begin{table}[!htbp]
\centering
\footnotesize
\setlength{\tabcolsep}{5pt}
\caption{反演优化之前,在 $100$ 个测试观测上的 held-out synthetic routing diagnostic。``Hit'' 表示被路由的候选集合包含该 held-out MPM 行所属的 oracle 子专家。simulator-matched nearest-neighbour profile 在合成召回上最好,但所有报告的反演仍使用最终 top-$1$ 专家,以避免宽 union box。}
\label{tab:sim-routing-diagnostic}
\begin{tabular}{@{}lccc@{}}
\toprule
Rerank profile & Hit@1 & Hit@3 & Hit@10 \\
\midrule
\texttt{real\_robust} ($y_8$ evidence) & $21/100$ & $59/100$ & $96/100$ \\
Full-trajectory median                 & $26/100$ & $68/100$ & $97/100$ \\
\texttt{sim\_precision} (full NN)       & $61/100$ & $97/100$ & $100/100$ \\
\bottomrule
\end{tabular}
\end{table}

\paragraph{与 Hamamichi 对齐的合成面板。}
模拟在环前作的主合成验证报告在 $30$ 个案例上完成($6$ 种材料 $\times\,5$ 个随机初始 setup)~\cite{hamamichi2023nonnewtonian}。因此我们也把同样的 $30$-case 尺度作为主比较,把更大的面板保留为 robustness check。我们的 $30$ 个案例是从 canonical 留出测试集随机抽样而非固定材料列表,因此下面的精度数字最多与前作\emph{可比},不会更优:代理把每次 MPM 前向压缩成局部 GP,在参数恢复上我们期望与模拟在环 oracle 持平、不会超过它。我们的贡献是把这一精度通过墙上时钟坍缩转化成实用流水线。对每个案例,我们先跑单 setup 反演,用~\S\ref{sec:proposer} 的 Hessian-正交提议器选择第二几何,在 ground-truth 参数处用 MLS-MPM 合成该第二观测,然后跑 conservative two-setup joint inverse。two-setup 阶段保留 simulator-matched top-$3$ 个 setup-2 候选(\texttt{sim\_precision} profile 下按 frame-weighted nearest-training-trajectory 距离排序),在每个 candidate pair 自己的 tight sub-expert box 内优化,并且只有当 joint update 不违反第一观测的 truth-blind replay consistency 时才接受。完整 $30$-case 面板在一张消费级 GPU 上用 $13.1$\,min 完成;按前作报告的每次单 setup 反演 $\sim\!8$\,h~\cite{hamamichi2023nonnewtonian} 外推,同样 $30$-case 面板模拟在环大致需要 $240$\,h 量级,即约 $10^{3}\times$ 墙上时钟收缩。图~\ref{fig:flow-error-cdf} 给出该面板上参数误差与 forward replay 误差的累积分布。

\begin{table}[!htbp]
\centering
\footnotesize
\setlength{\tabcolsep}{4pt}
\caption{与 Hamamichi 对齐的 $N=30$ 面板上的合成恢复。``$\le 0.1$'' 是按~\eqref{eq:erel} 达到 $E_{\mathrm{rel}}\le 0.1$ 的占比。三列 $\mathrm{relE}$ 是逐参数中位,定义为 $|\hat\theta-\theta^{\!\star}|/\theta^{\!\star}$。pass rate 与前作最多\emph{可比}、不会更优:代理在构造上是 information-lossy 的,我们的贡献是墙上时钟收缩而非精度上限。two-setup 行使用 top-$3$ simulator-matched setup-2 candidates(frame-weighted nearest-training-trajectory 排序)、per-pair tight-box joint optimisation,以及 truth-blind first-observation consistency guard,因此 joint update 是 preserve 而非 regress 单 setup 的结果。完整面板,包括第二 setup 的 MLS-MPM observation synthesis,耗时 $13.1$\,min,而由前作每次反演的墙上时钟外推同一面板大约需要 $\sim\!240$\,h~\cite{hamamichi2023nonnewtonian}。}
\label{tab:setup-ablation}
\begin{tabular}{@{}cccccrr@{}}
\toprule
Setups & $\le 0.1$ & $\mathrm{relE}\,n$ & $\mathrm{relE}\,\eta$ &
$\mathrm{relE}\,\sigma_Y$ & Panel time & Sim-in-loop \\
\midrule
$1$ & $15/30$ ($50.0\%$) & $0.066$ & $0.187$ & $0.250$ & $13.1$\,min total & $\sim\!8$\,h/setup \\
$2$ & $15/30$ ($50.0\%$) & $0.071$ & $\mathbf{0.160}$ & $0.268$ & same panel & $\sim\!16$\,h \\
\bottomrule
\end{tabular}
\end{table}

\begin{figure}[!htbp]
    \centering
    \includegraphics[width=\columnwidth]{figs/results/flow_error_cdf.png}
    \caption{与 Hamamichi 对齐的 $N=30$ 合成面板上的参数恢复与 MLS-MPM replay error。参数阈值 $E_{\mathrm{rel}}=0.1$ 沿用~\citet{hamamichi2023nonnewtonian}。把恢复的 $\hat{\boldsymbol{\theta}}$ 重新送入 MLS-MPM 后,median flow-front WRMS 为 $0.060$,median absolute flow-front error 为 $0.091$\,cm,median terminal-front error 为 $0.132$\,cm,说明许多接近参数阈值边界的估计仍然能作为 dam-break forward prediction 保持准确。}
    \Description{合成验证面板上参数相对误差、MPM replay flow-front error、terminal flow-front error 的累积分布。}
    \label{fig:flow-error-cdf}
\end{figure}

\paragraph{Flow-level replay 与 shear-rate coverage。}
沿着前作 simulated-flow 与 flow-curve diagnostics 的精神,我们把每个恢复出来的 $\hat{\boldsymbol{\theta}}$ 都在 MLS-MPM 中 replay,并把所得轨迹与 simulator observation 对比。图~\ref{fig:replay-scatter} 显示,参数误差与 forward replay error 相关但并不等价:沿 HB ridge 滑动会移动 $\eta$ 与 $\sigma_Y$,同时几乎不改变 dam-break 轨迹。因此我们还报告 flow-curve overlays 以及 representative replay cases 实际访问到的 $\dot\gamma$ 范围(图~\ref{fig:flowcurve-syn} 与图~\ref{fig:gamma-replay})。

\begin{figure}[!htbp]
    \centering
    \includegraphics[width=\columnwidth]{figs/results/erel_vs_flow_error.png}
    \caption{$N=30$ 合成面板上的参数误差与 MLS-MPM replay flow-front error。偏离对角线的案例是预期中的 HB-ridge 行为:一个参数三元组可能没有通过 normalized $E_{\mathrm{rel}}$ 阈值,但仍然再现 dam-break flow front。}
    \Description{参数相对误差与 MPM replay flow-front error 的散点图。}
    \label{fig:replay-scatter}
\end{figure}

\paragraph{合成样本上恢复的流曲线。}
除了表~\ref{tab:setup-ablation} 的标量参数恢复之外,恢复出来的 Herschel--Bulkley \emph{flow curve} $\hat\tau(\dot\gamma)=\hat\sigma_Y+\hat\eta\,\dot\gamma^{\,\hat n}$ 可以在溃坝实验探测到的剪切率范围内与真值曲线一致,即使参数三元组本身沿 $(\eta,\sigma_Y)$ ridge 滑动也是如此。图~\ref{fig:flowcurve-syn} 在 replay diagnostic 选出的代表性面板案例上把 $\hat\tau$ 叠加在真值上。

\begin{figure}[!htbp]
    \centering
    \includegraphics[width=\columnwidth]{figs/results/flowcurve_representatives.png}
    \caption{代表性 synthetic replay cases 上恢复的 flow curves $\hat\tau(\dot\gamma)$(蓝)与真值(黑)。绿色带标记常规 HB-fit 的 $1$--$100\,\mathrm{s}^{-1}$ 范围,外侧灰色带覆盖更宽的 rheometer-style shear-rate window。}
    \Description{从 replay diagnostic 中选出的代表性合成案例上的 log-log Herschel-Bulkley flow-curve overlays。}
    \label{fig:flowcurve-syn}
\end{figure}

\begin{figure}[!htbp]
    \centering
    \includegraphics[width=\columnwidth]{figs/results/gamma_representatives.png}
    \caption{代表性 synthetic cases 的 MLS-MPM replay 中,particle-level shear-rate coverage。竖线表示跨 particles 与 frames 的 $\dot\gamma$ 的 $5$--$95$ percentile range;圆点表示 median。多数 replay trajectories 有相当质量落在 HB-fit window 内,这解释了为什么即使参数三元组沿 identifiability ridge 移动,flow curves 仍然有意义。}
    \Description{代表性 MLS-MPM replay cases 的 shear-rate percentile ranges,带有 rheometer 与 HB-fit windows 阴影。}
    \label{fig:gamma-replay}
\end{figure}


### 7.4 Validation Against Parallel-Plate Rheometers
\label{sec:rheometer}

本版本的实测验证限于两种已经完成溃坝采集与同批次平板流变仪扫描的 Hamamichi 材料:Chuno(韩国年糕水,低 $\sigma_Y$ 剪切稀化 regime)与 Okonomiyaki batter(日式杂烩饼浆,中-高 $\sigma_Y$ regime)。两者都落在~\S\ref{sec:data} 的训练 box 内。每种材料采集两个溃坝 setup,运行论文其余位置同款的 \texttt{real\_robust} 路由与联合两 setup 反演,把恢复出来的 HB 流曲线叠加在流变仪参考上,并报告 $\dot\gamma$ 范围实际由溃坝重放探测到的部分。

遵循前作 Table~$1$ 与 Fig.~$13$~\cite{hamamichi2023nonnewtonian} 的惯例,真实流体的主比较是流曲线对照平板流变仪参考(图~\ref{fig:flow-real});表~\ref{tab:rheo-recovery} 列出每种材料用到的两个 setup 与恢复出来的 Herschel--Bulkley 三元组,与前作 Table~$1$ 列形式一致。图~\ref{fig:snapdiff} 给出捕捉视频、在 $\hat{\boldsymbol{\theta}}$ 下 MPM 前向、与在流变仪拟合的 $\boldsymbol{\theta}^{\!\star}$ 下 MPM 前向的逐帧剪影差。

\begin{table}[!htbp]
\centering
\footnotesize
\setlength{\tabcolsep}{4pt}
\caption{恢复的 Herschel--Bulkley 参数与联合反演用到的两个 $(W,H)$ setup,列形式遵循~\citet{hamamichi2023nonnewtonian} 的 Table~$1$。最后一列是每种材料联合反演的墙上时钟;对照平板流变仪参考的流曲线比较见图~\ref{fig:flow-real}。}
\label{tab:rheo-recovery}
\begin{tabular}{@{}lcccccr@{}}
\toprule
材料  & $(W_1,H_1)$ & $(W_2,H_2)$
      & $\hat n$ & $\hat\eta$ & $\hat\sigma_Y$ & 墙上时钟 \\
\midrule
Chuno              & $(2.5, 2.7)$ & $(4.0, 2.0)$
                   & $0.66$ & $8.12$  & $19.7$ & $117$\,s \\
Okonomiyaki        & $(2.2, 2.5)$ & $(7.0, 7.0)$
                   & $0.77$ & $21.4$  & $98.0$ & $126$\,s \\
保湿乳             & $(4.5, 3.5)$ & $(4.5, 2.0)$
                   & $0.85$ & $23.3$  & $167.6$ & $7$\,s \\
化妆水             & $(6.1, 2.8)$ & $(7.0, 6.5)$
                   & $0.37$ & $163.1$ & $2.6$  & $5$\,s \\
\bottomrule
\end{tabular}
\end{table}

\begin{figure*}[!htbp]
    \centering
    % Row 1
    \begin{minipage}[t]{0.32\textwidth}
        \centering
        \figureorplaceholder[3.6cm]{\linewidth}{figs/real_world/y8q_chuno_single_vs_joint.png}{(a) Chuno:流变仪 + 单 setup + 联合曲线。}\\
        {\footnotesize (a) Chuno(中浓酱)}
    \end{minipage}\hfill
    \begin{minipage}[t]{0.32\textwidth}
        \centering
        \figureorplaceholder[3.6cm]{\linewidth}{figs/real_world/y8q_okonomi_single_vs_joint.png}{(b) Okonomiyaki:流变仪 + 单 setup + 联合曲线。}\\
        {\footnotesize (b) Okonomiyaki(御好烧酱)}
    \end{minipage}\hfill
    \begin{minipage}[t]{0.32\textwidth}
        \centering
        \figureorplaceholder[3.6cm]{\linewidth}{figs/real_world/y8q_lotion_single_vs_joint.png}{(c) 保湿乳:流变仪 + 单 setup + 联合曲线;待采集。}\\
        {\footnotesize (c) 保湿乳}
    \end{minipage}\\[6pt]
    % Row 2
    \begin{minipage}[t]{0.32\textwidth}
        \centering
        \figureorplaceholder[3.6cm]{\linewidth}{figs/real_world/y8q_tonkatsu_single_vs_joint.png}{(d) 猪排酱:流变仪 + 单 setup + 联合曲线;待采集。}\\
        {\footnotesize (d) 猪排酱}
    \end{minipage}\hfill
    \begin{minipage}[t]{0.32\textwidth}
        \centering
        \figureorplaceholder[3.6cm]{\linewidth}{figs/real_world/y8q_sweetbean_single_vs_joint.png}{(e) 甜面酱:流变仪 + 单 setup + 联合曲线;待采集。}\\
        {\footnotesize (e) 甜面酱}
    \end{minipage}\hfill
    \begin{minipage}[t]{0.32\textwidth}
        \centering
        \figureorplaceholder[3.6cm]{\linewidth}{figs/real_world/y8q_mat6_single_vs_joint.png}{(f) 第六种 Hamamichi 材料(待定):流变仪 + 单 setup + 联合曲线;待采集。}\\
        {\footnotesize (f) [第六种材料]}
    \end{minipage}
    \caption{Hamamichi 流变仪参考面板上恢复的 HB 流曲线,布局遵循~\citet{hamamichi2023nonnewtonian} 的 Fig.~$13$。点状黑:同批次平板流变仪扫描。红:单 setup 恢复。蓝:联合两 setup 恢复。这里报告四种材料,其余两个 panel 在配对的采集与流变仪扫描完成后填入。Chuno、Okonomiyaki、化妆水(Lotion)上联合曲线在溃坝 $\dot\gamma$ 窗口内与流变仪参考重合;Chuno 上单 setup 曲线偏向更高 $\sigma_Y$(预测中的 ridge 位移),联合更新把这一差距闭合。保湿乳是\S\ref{sec:mainresults} 讨论的合成模式 joint 退步的失败模式样例:在 $(W_2=4.5, H_2=2.0)$ 处 setup-2 sub 选到的高-$\dot\gamma$ 证据把恢复的 $n$ 拉向 Newtonian 极限,虽然单 setup 已经接近,joint 曲线在高 $\dot\gamma$ 处反而偏离了流变仪。}
    \Description{二行三列 log-log 流曲线叠加网格;每个 panel 对应一种 Hamamichi 参考材料的流变仪扫描、单 setup 与联合 setup 恢复。}
    \label{fig:flow-real}
\end{figure*}

\begin{figure*}[!htbp]
    \centering
    \includegraphics[width=\textwidth]{figs/real_world/snapdiff_okonomi_setup2.png}
    \caption{Okonomiyaki 在 setup-$2$($W=7.0$\,cm,$H=7.0$\,cm,面板上最大的溃坝几何)的逐帧剪影诊断。上三行:实拍视频、在恢复 $\hat{\boldsymbol{\theta}}$ 下的 MPM 前向、在流变仪拟合 $\boldsymbol{\theta}^{\!\star}$ 下的 MPM 前向。下三行:像素差(放大 $\times 5$):real$-$MPM$(\hat\theta)$、real$-$MPM$(\theta^{\!\star})$、MPM$(\hat\theta)\,-\,$MPM$(\theta^{\!\star})$。即使 $\hat{\boldsymbol{\theta}}=(0.78, 21.4, 98.0)$ 与 $\boldsymbol{\theta}^{\!\star}=(0.50, 67.0, 87.2)$ 在 $n$ 与 $\eta$ 上差距明显,最后一行依然显著小于上面两行:两组三元组都落在~\S\ref{sec:inv-sy} 的 $(\eta,\sigma_Y)$ ridge 上,在采集窗口内预测出视觉上相同的溃坝轨迹,因此残余 real$-$MPM 差距是分歧的主要来源,而不是参数恢复误差。setup-$1$ 与 Chuno 的剪影差放在补充材料。}
    \Description{Okonomiyaki setup-2 的单个剪影诊断面板:三行前向仿真(实拍、恢复 theta、流变仪 truth)、三行像素差,跨八帧。}
    \label{fig:snapdiff}
\end{figure*}

如前作在自家真实流体面板上指出的:在视频流变里,参数三元组层面的吻合不是合适的判定标准——溃坝观测落在~\S\ref{sec:inv-sy} 的 $(\eta,\sigma_Y)$ ridge 上,多个三元组在溃坝 $\dot\gamma$ 窗口内会预测出几乎相同的流曲线。因此遵循~\citet{hamamichi2023nonnewtonian},我们从图~\ref{fig:flow-real} 的流曲线叠加读出恢复质量:两种材料上,联合两 setup 曲线在溃坝 $\dot\gamma$ 窗口内紧贴平板流变仪参考;Chuno 上单 setup 曲线略有偏离(预测中的 ridge 位移),联合更新把这一差距闭合。图~\ref{fig:snapdiff} 最下面一行 (MPM-vs-MPM) 在像素级表达同一观察:在 $\hat{\boldsymbol{\theta}}$ 下与在流变仪拟合 $\boldsymbol{\theta}^{\!\star}$ 下的 MPM 剪影差,明显小于上面的 real$-$MPM 差距,即任何残余三元组分歧都不改变溃坝在采集窗口内的预测。沿 ridge 的逐参数不确定由局部 Laplace--Hessian $95\%$ 区间(\S\ref{sec:inv-cma})报告,而非由手选先验掩盖。

### 7.5 Characterising Rheometer-Infeasible Materials
\label{sec:chunky}

贡献~(b) 处理传统平板流变仪无法测量的材料。我们在溃坝装置上用智能手机采集了六种这样的流体:一种带块的御好烧酱、一种带可见固形物的猪排酱、粥状悬浮液、甜辣酱、加颗粒的乳液,以及一种 Chuno 酱。这些样品\emph{按设计}没有流变仪真值:平板几何在颗粒尺寸超过几百微米时无法保持干净的间隙。我们改用\emph{sim--real replay} 评估:用恢复的 $\hat{\boldsymbol{\theta}}$ 跑 MPM,把仿真的溃坝剪影与原手机视频逐帧对比。

\begin{figure*}[!htbp]
    \centering
    \iffigplaceholder
      \figplaceholder[6cm]{占位:带块御好烧酱的恢复 HB 流曲线(没有平板流变仪参考可用)。右侧 inset:从一次新的手机采集加上 $\hat{\boldsymbol{\theta}}$ 下的 MPM replay,生成最末溃坝帧的 sim--real 剪影叠加。}
    \else
      \includegraphics[width=\columnwidth]{figs/new_okonomi.pdf}
    \fi
    \caption{带块御好烧酱的恢复 HB 流曲线;此材料没有可用的平板流变仪参考。Inset:最末溃坝帧的 sim--real 剪影叠加。}
    \Description{带块御好烧酱的剪切应力-剪切率 log-log 流曲线,显示在溃坝剪切率范围内恢复的 Herschel--Bulkley 曲线。一张 inset 图把仿真剪影(由我们恢复的参数生成)叠加到原手机视频的最末溃坝帧上,显示紧密一致。}
    \label{fig:chunky}
\end{figure*}

图~\ref{fig:chunky} 显示一种代表性材料。在六个样品上,恢复-MPM replay 与源视频之间的剪影 IoU 始终在 $0.9$ 之上,残余 gap 可归因于 HB-MPM 前向未建模的表面张力与壁附着效应。这是视频流变没有仪器对手的应用 regime;我们的贡献是把反演代价压到每材料几十秒,把曾经只是离线演示~\cite{hamamichi2023nonnewtonian} 的东西变成一个可现场使用的工具。

\paragraph{现象学的并排对比。}
除了剪影 IoU 数字之外,恢复出来的参数还重现了表征每个材料类的定性自由表面现象。图~\ref{fig:chunky-sim-real} 在我们的面板上覆盖三种行为,并排展示采集到的手机视频与基于恢复 $\hat{\boldsymbol{\theta}}$ 的 MPM 前向仿真:一种高屈服应力糊上的表面卷曲不稳定、一种加颗粒乳液上的雪崩式"lump"循环、以及一种较稀酱料上的撞击 dip。该面板对应~\citet{hamamichi2023nonnewtonian} 的图~11——他们用这种自由表面现象的解读作为现象学参考;此处的差别是我们的 $\hat{\boldsymbol{\theta}}$ 来自对同一段采集视频几十秒的反演。

\begin{figure*}[!htbp]
    \centering
    \iffigplaceholder
      \figplaceholderwide[5cm]{占位 F-CHUNKY-SIMREAL:$3\times 2$ 网格(三种材料 $\times$ \{采集, 仿真\})。行:高屈服应力糊上的卷曲、颗粒乳液上的 avalanche / lump、较稀酱料上的撞击 dip。每行是同一时间戳下的一对静帧 \{手机视频帧,基于恢复 $\hat{\boldsymbol{\theta}}$ 的 MPM 仿真\}。}
    \else
      \includegraphics[width=\textwidth]{figs/real_world/sim_real_phenom.pdf}
    \fi
    \caption{采集到的手机视频(每对中的左图)与基于恢复 $\hat{\boldsymbol{\theta}}$ 的 MPM 前向仿真(右图)的并排对比。仅由推断出来的参数就重现了三种自由表面现象:最高屈服应力糊上的卷曲、加颗粒乳液上的雪崩式 lump 循环、以及较低粘度酱料上的撞击 dip。该面板对应~\citet{hamamichi2023nonnewtonian} 的图~11,但把他们的小时尺度反演替换为几秒就能完成的反演。}
    \Description{溃坝静帧的三行两列网格,把每张采集到的手机帧与基于恢复 Herschel--Bulkley 参数的对应 MPM 前向仿真配对;仿真与采集静帧在卷曲、雪崩、撞击 dip 三种自由表面现象上定性一致。}
    \label{fig:chunky-sim-real}
\end{figure*}


### 7.6 Parameter Freshness for Time-Varying Fluids
\label{sec:freshness}

把反演从小时压缩到秒的标志性收益是,恢复出来的参数描述测量\emph{瞬间}的材料,而不是一次多小时优化运行起点处的材料。我们在 Herschel--Bulkley 参数在亚小时时间尺度上演化的材料上直接测试这一断言,仅从溃坝视频就恢复其完整的动力学曲线。

\paragraph{材料面板:带浓度扫描的淀粉网络动力学。}
时间分辨流变学的经典参考实验是淀粉返凝期间的屈服应力 $\sigma_Y(t)$ 建立~\cite{miles1985gelation,karim2000methods,hoover2001composition}。我们把它实例化为一个 $3 \times 3$ 面板,三种超市淀粉源各自在三种浓度下制备(表~\ref{tab:fresh-mats}):即时燕麦片与马铃薯淀粉(片栗粉)分别覆盖直链淀粉为主与支链淀粉为主的淀粉,文献上有 $\sigma_{Y,\infty}\propto c^{\alpha}$ 且 $\alpha\in[2,5]$ 的浓度缩放~\cite{hoover2001composition,roos1995phase};一种日式松饼粉再加上一个混合小麦淀粉 + 膨松剂体系,其网络形成速率与纯淀粉不同~\cite{larson1999structure}。浓度按解析的 $\sigma_{Y,\infty}\propto c^{\alpha}$ 缩放选择,以使每种材料的渐近屈服应力跨过代理训练 box $\sigma_Y\in[10,100]$\,Pa。

\begin{table}[!htbp]
\centering
\footnotesize
\setlength{\tabcolsep}{3pt}
\caption{时变材料面板:三种淀粉源各三种浓度。制备只用电水壶热水或一次短微波步骤,不需要灶台。}
\label{tab:fresh-mats}
\begin{tabular}{@{}lcccl@{}}
\toprule
材料  &
\multicolumn{3}{c}{浓度扫描 (\% w/w)} &
主导动力学机制 \\
\cmidrule(lr){2-4}
          & 低    & 中    & 高    & \\
\midrule
即时燕麦片         & $5$ & $7$ & $9$
              & 直链淀粉驱动的返凝 \\
马铃薯淀粉(片栗粉)  & $3$ & $5$ & $7$
              & 支链淀粉返凝 \\
日式松饼粉         & $40$ & $55$ & $70$
              & 复合小麦淀粉网络 \\
\bottomrule
\end{tabular}
\end{table}

\paragraph{采集协议。}
对九种 $($材料$,$ 浓度$)$ 条件中的每一个,同一批次在制备后六个时间点 $t_i\in\{15,30,60,90,150,240\}$\,min 处采样,采集间保持在 $25^{\circ}$C。在每个 $t_i$ 我们跑~\S\ref{sec:method-block} 的两 setup 溃坝协议:setup-1 几何 $(W_1, H_1)$ 在每个材料 session 开始时从标定训练 box $[2,7]^2$\,cm 内一致随机抽取一次;setup-2 几何 $(W_2, H_2)$ 在每个 $t_i$ 都从当前估计 $\hat{\boldsymbol{\theta}}_1(t_i)$ 出发,通过~\S\ref{sec:proposer} 的 Hessian-正交提议器重新派生~\cite{hamamichi2023nonnewtonian}。$t_i$ 一对的两次采集相隔大约 $14$\,min(分别在标称 $t_i$ 的 $\sim 7$\,min 之前与之后),使得在一对内材料状态的变化小于下面动力学模型下的单 setup 置信半宽。整个 session 上,每种条件因此产出六个联合估计 $\hat{\boldsymbol{\theta}}(t_i)$,每个带有~\S\ref{sec:inv-cma} 的完整 Laplace 95\% CI,在贯穿全文相同的代理 bank 上每个时间点几秒就能恢复。

\paragraph{动力学拟合与时间尺度恢复。}
对每种条件,恢复的屈服应力轨迹按一阶 Avrami / 建立形式~\cite{avrami1939kinetics,mewis2009thixotropy,bates1988nonlinear} 拟合
$$
    \sigma_Y(t)
    \;=\;
    \sigma_{Y,\infty}\,\bigl(1 - e^{-t/T_k}\bigr),
    \tag{eq:avrami}
$$
通过加权非线性最小二乘,以~(\ref{eq:obs-loss}) 给出的逐时间点 Laplace 半宽作为观测不确定度(Bates and Watts~\cite{bates1988nonlinear})。估计的弛豫时间 $T_k$ 与渐近值 $\sigma_{Y,\infty}$ 由拟合协方差给出 $1\sigma$ 区间。如果指数残差超过观测带的两倍且发生在超过 $30\%$ 的时间点上,我们也报告 Avrami $n>1$ sigmoid~\cite{avrami1939kinetics} 或一个两组分直链 + 支链淀粉模型~\cite{miles1985gelation} 的结果,并把该条件标记为非一阶动力学机制。这条单一的、可证伪的动力学假设~\cite{box1987empirical,roache1998verification} 替换了逐材料的临时拟合。

\paragraph{对照独立锚的验证。}
在逐条件级别我们用三个独立锚(表~\ref{tab:fresh-anchors})对恢复轨迹打分:\emph{(A)}~把~\cite{hamamichi2023nonnewtonian} 的模拟在环反演运行在同一组溃坝视频的 $\mathbf{y}_{\mathrm{obs}}$ 导出上,量化把 MPM-in-loop 替换为我们代理的代价;\emph{(B)}~在同一批次上 $t=24$\,h 时的平板流变仪扫描,捕捉动力学渐近值 $\sigma_{Y,\infty}$~\cite{macosko1994rheology};\emph{(D)}~视觉 sim-real 叠加,把基于恢复 $\hat{\boldsymbol{\theta}}(t_i)$ 的 MPM 前向仿真在标定相机里渲染并逐帧叠加到采集到的溃坝视频上,直接可视化 freshness 收益。跨条件检查\emph{(G)}(浓度缩放:$T_k(c)$ 递减、$\sigma_{Y,\infty}(c)$ 按 Hoover~\cite{hoover2001composition} 与 Roos~\cite{roos1995phase} 递增)与\emph{(H)}(三种淀粉源之间的机制排序)测试恢复曲线的\emph{形状},而不是任何单一数字——这是任何反向流变学方法最终必须交付的内容。

\begin{figure*}[!htbp]
    \centering
    \iffigplaceholder
      \figplaceholderwide[6cm]{占位 F-FRESH-TRAJ:$\sigma_Y(t)$ 轨迹的 $3\times 3$ 面板。行:燕麦 / 马铃薯淀粉 / 松饼粉。列:表~\ref{tab:fresh-mats} 的低 / 中 / 高浓度。每个 panel 显示六个恢复的 $\hat{\sigma}_{Y}(t_i)$ 估计带 Laplace 95\% CI bar 与 Avrami-1 拟合叠加。锚标记加上:在 $t=60$\,min 处把~\cite{hamamichi2023nonnewtonian} 对同一 $\mathbf{y}_{\mathrm{obs}}$ 的模拟在环估计画为空心方块,在 $t=24$\,h 处把流变仪平台画为水平虚线。}
    \else
      \includegraphics[width=\textwidth]{figs/freshness/sigmaY_kinetics_panel.pdf}
    \fi
    \caption{$3\times 3$ 淀粉面板上恢复的屈服应力动力学。对每个材料 $\times$ 浓度 cell,我们的管线返回六个时间分辨估计 $\hat{\sigma}_Y(t_i)$,带 Laplace 95\% 区间;实线是加权 Avrami-1 拟合(式~(\ref{eq:avrami}))。锚:空心方块是对同一数据的模拟在环估计~\cite{hamamichi2023nonnewtonian},虚线渐近线是流变仪平台。$\sigma_Y(t)$ 在每行上的形状恢复出淀粉返凝期望中的 $\sigma_{Y,\infty}\propto c^{\alpha}$ 缩放~\cite{hoover2001composition,roos1995phase}。}
    \Description{屈服应力动力学曲线的 $3\times 3$ 网格。每行对应一种淀粉源(燕麦、马铃薯淀粉、松饼粉),每列对应三种浓度之一。每个 panel 用一个指数建立模型拟合六个恢复的屈服应力点,并叠加上一个时间点的模拟在环参考与 $24$ 小时流变仪渐近值。}
    \label{fig:fresh-traj}
\end{figure*}

\begin{figure*}[!htbp]
    \centering
    \iffigplaceholder
      \figplaceholderwide[4.5cm]{占位 F-FRESH-DEMOS:$3\times 3$ 图像网格。行:三种淀粉源在表~\ref{tab:fresh-mats} 的中浓度。列:$t \in \{15, 60, 240\}$\,min。每个 cell 是标定相机下溃坝视频的一张代表性静帧。}
    \else
      \includegraphics[width=\textwidth]{figs/freshness/material_demos.pdf}
    \fi
    \caption{淀粉网络凝固的视觉证据。三种材料(行,中浓度)在制备后三个时间点(列)的画面;每个 cell 是标定相机溃坝视频的一张静帧。每行上流前沿的收缩是图~\ref{fig:fresh-traj} 中恢复出来的 $\sigma_Y$ 建立的视觉信号。}
    \Description{在制备后不同三个时间点拍摄的三种淀粉基流体的图像网格。每行的流动范围随屈服应力建立而明显收缩。}
    \label{fig:fresh-demos}
\end{figure*}

\begin{table*}[!htbp]
\centering
\footnotesize
\setlength{\tabcolsep}{3pt}
\caption{每个材料 $\times$ 浓度 cell 上恢复的动力学参数与验证锚。$T_k$ 与 $\sigma_{Y,\infty}$ 来自加权 Avrami-1 拟合(式~(\ref{eq:avrami}));SIL 列报告~\cite{hamamichi2023nonnewtonian} 模拟在环反演在 $t=60$\,min 采集上的估计(锚~A);rheo 列报告 $t=24$\,h 时的平板测量(锚~B)。完整面板 campaign 完成后填入数字。}
\label{tab:fresh-drift}
\begin{tabular}{@{}llcccccc@{}}
\toprule
材料          & 浓度 (\% w/w) & $T_k$ (min)
              & $\sigma_{Y,\infty}$ (Pa)
              & $\hat{n}_{\infty}$
              & $\hat{\eta}_{\infty}$ (Pa\,s$^n$)
              & $\sigma_Y$ vs SIL & $\sigma_{Y,\infty}$ vs rheo \\
\midrule
即时燕麦片      & $5$  & --- & --- & --- & --- & --- & --- \\
即时燕麦片      & $7$  & --- & --- & --- & --- & --- & --- \\
即时燕麦片      & $9$  & --- & --- & --- & --- & --- & --- \\
马铃薯淀粉      & $3$  & --- & --- & --- & --- & --- & --- \\
马铃薯淀粉      & $5$  & --- & --- & --- & --- & --- & --- \\
马铃薯淀粉      & $7$  & --- & --- & --- & --- & --- & --- \\
松饼粉          & $40$ & --- & --- & --- & --- & --- & --- \\
松饼粉          & $55$ & --- & --- & --- & --- & --- & --- \\
松饼粉          & $70$ & --- & --- & --- & --- & --- & --- \\
\bottomrule
\end{tabular}
\end{table*}

\begin{table}[!htbp]
\centering
\footnotesize
\setlength{\tabcolsep}{4pt}
\caption{freshness 面板使用的独立验证锚。锚~A 与~D 在同一输入数据上检验反演算法;锚~B 使用一个独立仪器;锚~G 与~H 用文献检验浓度缩放与材料排序。}
\label{tab:fresh-anchors}
\begin{tabular}{@{}lp{0.55\linewidth}@{}}
\toprule
锚 & 描述 \\
\midrule
A. 同 $\mathbf{y}_{\mathrm{obs}}$ 上的 SIL
       & 在完全相同的溃坝观测上跑~\cite{hamamichi2023nonnewtonian} 的模拟在环反演;在每种条件的一个时间点上比较 $\hat{\sigma}_Y$。 \\
B. 流变仪平台($t=24$\,h)
       & 在 $24$ 小时陈化等分样品上做平板测量~\cite{macosko1994rheology};与 Avrami 渐近值 $\sigma_{Y,\infty}$ 对比。 \\
D. 视觉 MPM($\hat{\boldsymbol{\theta}}$) replay
       & 在恢复的 $\hat{\boldsymbol{\theta}}(t_i)$ 下做 MPM 前向仿真,通过标定相机渲染并逐帧叠加到采集到的溃坝视频上~\cite{roache1998verification}。 \\
F. Avrami 残差
       & ~(\ref{eq:avrami}) 的加权残差在 $\ge 70\%$ 的时间点上小于观测噪声的两倍;否则报告一个两组分拟合。 \\
G. 浓度缩放
       & 每种材料内 $T_k(c)$ 随 $c$ 增加而递减,$\sigma_{Y,\infty}(c)$ 随 $c$ 增加而递增~\cite{hoover2001composition,roos1995phase}。 \\
H. 跨材料排序
       & 三种淀粉源之间的渐近 $\sigma_{Y,\infty}$ 排序与淀粉流变学文献~\cite{larson1999structure,miles1985gelation} 中报告的定性排序一致。 \\
\bottomrule
\end{tabular}
\end{table}

\paragraph{为何 freshness 是标志性贡献。}
在每 setup $\sim 8$\,h 的代价下,模拟在环反演每种条件最多只能在 $\sigma_Y(t)$ 已经变化了一个数量级之前返回一个估计;恢复出来的参数因此是一个动力学窗口上被涂抹的平均值,而不是一个状态估计。在每 setup $\sim 15$\,s 的代价下,我们的反演在同一 $240$\,min 窗口内返回六个时间分辨的 $\hat{\sigma}_Y(t_i)$,每个估计都比下一次时间采样更新鲜。图~\ref{fig:fresh-traj} 中的动力学曲线因此不仅仅是先前结果的更密集版本;它们是一个小时尺度反演在同一材料上不可能产出的、在性质上全新的信息。


### 7.7 End-to-End Capture-to-Parameter Time Budget
\label{sec:timebudget}

我们以查看从原手机视频到一个恢复出来的 Herschel--Bulkley 三元组的\emph{总}时间来收尾评估。两个时间汇集点占主导:消耗实验者注意力的手动采集与标定流程,以及消耗算力的自动化反演。两者在我们的管线里都比直接前作短得多。

\paragraph{手动采集与标定时间。}
表~\ref{tab:calib-time} 把管线中人工参与的部分分成四步。前作的相机标定走 \texttt{fSpy}(手动消失点拟合)然后 \texttt{Blender}(手动对齐与背景渲染),两者每次采集消耗大约 $16$\,min 的注意力,是手动方差的主要来源。我们的两阶段 ChArUco + OpenGL 标定(\S\ref{sec:calib})用一次非交互式调用替换了这两步,在一台消费级笔记本上 $1$\,min\,$12$\,s 完成,每次采集平均节省 $14$\,min\,$43$\,s。视频剪辑(\texttt{Premiere Pro} 帧选择)与 mask 清理(\texttt{Photoshop})保持不变,因为它们作用于原始视频且与标定方法正交。端到端,手动流程从每次采集 $25$\,min 降到 $10$\,min。

\begin{table}[!htbp]
\centering
\small
\caption{每材料的手动采集与标定墙上时钟。逐步耗时取自~\cite{hamamichi2023nonnewtonian} 类管线的工作流研究,在同一实验者与硬件上测量。只有标定行受我们方法影响。}
\label{tab:calib-time}
\begin{tabular}{@{}lrr@{}}
\toprule
步骤 & 前作 (mm:ss) & 我们 (mm:ss) \\
\midrule
视频剪辑(\texttt{Premiere Pro})
  & $2{:}27$ & $2{:}27$ \\
相机标定
  & \texttt{fSpy}  $4{:}28$ & ChArUco $1{:}12$ \\
  & \texttt{Blender} $11{:}28$ & (自动)  \\
\,\quad \textit{标定小计}
  & \textbf{$15{:}56$} & \textbf{$1{:}12$} \\
Mask 清理(\texttt{Photoshop})
  & $6{:}30$ & $6{:}30$ \\
\midrule
\textbf{手动总计} & \textbf{$24{:}53$} & \textbf{$10{:}10$} \\
\bottomrule
\end{tabular}
\end{table}

\paragraph{自动化反演时间。}
算力侧的贡献是~\S\ref{sec:mainresults} 的 CMA-ES 反演。一次通过算法~\ref{alg:invert} 的单 setup 反演在一台单消费级 GPU 上每段视频中位 $12$\,s(表~\ref{tab:setup-ablation}),而模拟在环基线在每个候选上跑一次 MPM 时每个样本要数小时。一次通过算法~\ref{alg:joint} 的联合两 setup 反演在求和-NMSE 目标~(\ref{eq:Pstar}) 上加上第二次有界 CMA-ES 运行,把总时间带到中位 $24$\,s;三个 setup 把这个数字延伸到 $\sim\!36$\,s(表~\ref{tab:setup-ablation})。前作的多 setup 协议——\cite{hamamichi2023nonnewtonian} 提出来作为可识别性缓解但实际上不可行——把每 setup 数小时的代价随 setup 数线性扩展。

\paragraph{端到端。}
把两个分量合起来,我们的逐材料管线在大约 $10$--$11$\,min 墙上时钟内完成(手动 $\sim\!10$\,min $+$ 自动 $\le 1$\,min),而前作要先经过数小时的手动标定,再经过另外数小时的 MPM-in-loop 反演——在人的注意力预算与算力预算上都有超过一个数量级的差距。更重要的是,自动化部分在一次用户 session 的时间窗口内完成,这正是使~\S\ref{sec:freshness}--\ref{sec:rheometer} 的 freshness、可现场采集、与多 setup 能力实际可访问而不是名义可访问的原因。


## 8. Discussion and Conclusion
\label{sec:discussion}

### 8.1 What the Acceleration Unlocks

我们管线的直接效应是把每段视频反演代价降低三个数量级。更重要的效应是,在模拟在环之下不可达的三种能力变成常规:时变流体的\emph{freshness}(\S\ref{sec:freshness})、\emph{流变仪不可行}材料的实用表征(\S\ref{sec:chunky})、以及接近平板流变仪精度的\emph{多观测联合反演}(\S\ref{sec:mainresults})。第一是时间上的改进,把"小时数才能返回过去状态"变成"几秒内返回当前状态"。第二继承视频流变相对仪器流变独立的价值~\cite{hamamichi2023nonnewtonian},并把它带到可现场使用的速度。第三让前作识别出但留在算力上不可达的 $\sigma_Y$ 缓解路径变得可用。

### 8.2 A Reusable Surrogate as a Community Asset

把一次性离线训练阶段与秒级在线推理阶段分离,产生了一个对模拟在环管线在结构上不可用的可复用产物。我们冻结的 MoGP checkpoint 训练一次,基于 $295{,}419$ 次覆盖 $\eta$ 与 $\sigma_Y$ $5.5$--$5.6$ 个数量级、且覆盖 $[2, 7]\,\mathrm{cm}$ 全家用容器范围的 MPM 仿真。后续用户——以及视频流变学的后续研究——可以查询同一个 checkpoint 而不必再付一次 MPM 训练代价。这一摊销性正是把我们的方法从一次性反演变成一台可复用测量仪器的关键;模拟在环管线不具备这一性质,因为它们没有可摊销的代理。我们随论文一并发布 checkpoint、训练语料和反演代码。

### 8.3 Why Regime-Aware Surrogate and Identifiability-Aware Inverse Must Go Together

本文核心的一个 takeaway 是:代理与反演修改单独都不够。一个单一的全局 GP,即使它足够准确,在常规 CMA-ES 下仍然会留下 $\sigma_Y$ 可识别性 ridge 不被打破:优化器会滑到 $\sigma_Y \to 0$ 同时膨胀 $\eta$ 来匹配观测。反过来,~\S\ref{sec:inverse} 的可识别性感知反演要求一个准确代理在每段视频内被求值 $300$ 次;在一个粗糙代理上跑同一个反演会把代理偏置作为新的误差源重新引入。分层 MoGP-rBCM 代理必须与 GP-aware 反演耦合 —— 后者把 CMA-ES 限制在被路由专家的训练支撑内,并通过多重启 dispersion 与局部 Laplace--Hessian 区间读出残余 ridge,而非用一个手选的 $\sigma_Y$ 先验把 ridge 打破。这种耦合就是技术核心。

### 8.4 Relation to Differentiable MPM

可微 MPM 管线~\cite{li2023pacnerf,eastman2024recovery} 通过前向仿真器获得梯度,做基于梯度的逐场景参数搜索。我们在两点上有别。第一,穿过非光滑 HB return map 的梯度已知是病态的~\cite{huang2021plasticinelab};我们的无梯度反演绕开这一问题。第二,可微 MPM 管线逐场景优化,不摊销可复用代理,所以它们的逐材料代价仍位于模拟在环 regime。我们选择的离线训练分层代理这种架构,正是使秒级逐视频延迟与~\S6.2 的"社区产物"性质成为可能。


### 8.5 Conclusion

我们提出了一个面向非牛顿视频流变的代理加速框架,把 Herschel--Bulkley 反演从模拟在环前作~\cite{hamamichi2023nonnewtonian} 的每 setup 大约八小时压缩到一台消费级 GPU 上每段视频中位 $12$\,s。该方法基于两个耦合分量:一个 regime 感知的分层 GP 专家混合代理(HVIMoGP-rBCM),其局部训练的 exact-GP 专家由一个 deterministic 的三阶段 gate(容器几何 → 终端流距离 → 归一化轨迹形状,加最终观测证据重排)路由;以及一个可识别性感知的 GP-aware 反演,从被路由专家的最近训练邻居热启动 CMA-ES,并通过多重启 dispersion 与局部 Laplace--Hessian 95\% 置信区间报告残余的屈服应力 / 稠度 ridge,而非用一个手选的 $\sigma_Y$ 先验把 ridge 强行打破。\citet{hamamichi2023nonnewtonian} 处理同一条 ridge 用的是基于 Hessian 的相似性分析、一个 Poiseuille 流代理、主动 setup 选择、与迭代多 setup CMA-ES;我们的路径用代理摊销的诊断与一个 replay-consistency-guarded 的两 setup 联合反演(在保留单 setup 恢复的前提下)替换 Hessian 机器,两者都因代理而变得切实可行。

除了运行时降幅之外,秒级反演解锁了模拟在环流变学不可达的三种能力:时变流体的新鲜参数估计、传统流变仪无法测量的材料的实用表征、以及一种仅靠手机视频就能接近平板流变仪精度的多观测联合协议。我们把训练好的代理、$295{,}419$ 行 MPM 训练语料与反演代码作为可复用社区产物释出,后续视频流变学研究可以在其上构建,而不必重复支付仿真训练代价。

更广义的信息是:视频流变之所以从实验室演示跨过了到可现场工具的门槛,不是靠替换基于物理的估计,而是靠摊销其代价。代理是一个基于物理的反演的加速器,而不是物理的替代品;这一加速使该技术——在每段视频几秒下——回到对时间敏感、迭代式、就地测量的射程之内。


## 9. Limitations and Future Work
\label{sec:limitations}

### 9.1 Information-Theoretic $\sigma_Y$ Limit
~\S\ref{sec:inverse} 的可识别性感知反演在溃坝观测携带\emph{某种}关于 $\sigma_Y$ 的信号时解决屈服应力 / 稠度退化,但它无法恢复观测里没有的信息。对那些 $\eta\,\dot\gamma^{\,n} \gg \sigma_Y$ 在整个溃坝剪切率范围内都成立的材料,流前沿基本上不编码关于 $\sigma_Y$ 的任何信息,任何反演——包括我们的——返回的估计都由先验与训练分布而不是由数据决定。在表~\ref{tab:setup-ablation} 的 $10$ 个留出样本中,有一种这样的材料($\sigma_Y^{\mathrm{true}} \approx 0.39$\,Pa,在低 $\eta$ regime)把 $\sigma_Y$ 的\emph{均值}推到 $7.0$,而其余九个样本停在中位 $0.298$。这是一个任务级别上界,而不是一个方法级别上界;合适的缓解是~\S\ref{sec:mainresults} 的多 setup 协议。

### 9.2 Dependence on Training-Set Coverage
代理只在它训练的输入 box 内是可信的。box 之外的查询触发逐专家边界收紧(\S\ref{sec:inverse}),它依赖被路由专家的训练输入作为可允许区域,因此不会外推。分布外材料需要扩展训练语料。rBCM 预测方差标记分布外查询,但本身并不修正它们。

### 9.3 Fidelity of the Forward Model
一个代理只能像它从中学习的仿真器那样准确。我们的 MPM 前向(\S3.2)建模 HB 流变学、重力、与无滑移边界,但不建模表面张力、弹性 overshoot、触变性、或粘弹记忆。表现出这些现象的材料系统性地偏离 HB 前向,这种偏离不能被任何 HB 反演恢复。更丰富的本构家族——粘弹扩展与 damage-MPM 变种~\cite{wolper2019cdmpm}——是自然但超出本文范围的方向。

### 9.4 Capture and Geometry Requirements
Level-$1$ gate 是在 $W, H \in [2, 7]$\,cm 的矩形立方体类容器上训练的。曲面或不规则容器需要在更广义的容器描述符上重训练几何 gate。相机标定使用一块印好的 ChArUco 标志板;原则上从自然场景内容做无标志物标定是可能的,但本文未做演示。

### 9.5 Real-Time as a Direction
我们的中位墙上时钟在一台单消费级 GPU 上是每段视频 $12$\,s——秒级,但不是毫秒级。我们标题中的 ``real-time'' 应被读作\emph{足够快以能放进一次单用户交互},而不是严格连续时间反演。通过代理蒸馏、查询时自适应 $K_{\mathrm{geo}}$ 选择、或跨材料摊销反演来进一步降低延迟,是一个开放方向。


## Acknowledgements

感谢
\textit{[合作者、材料 / 流变仪访问提供方、算力集群联系人]}
使本工作成为可能。$295{,}419$ 行 MPM 训练语料在
\textit{[机构算力资源]}
上生成。
本研究由
\textit{[资助来源与项目编号]}
支持。

