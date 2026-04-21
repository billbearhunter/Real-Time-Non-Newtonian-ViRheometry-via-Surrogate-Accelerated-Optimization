# 《面向实时的非牛顿视频流变测量:基于代理加速优化》

> **中文对照版**(非 submit 版本,仅供内部对照英文 `samplebody-journals.tex` 使用)
> 同步时间:2026-04-21
> 对应英文 commit:`5aa2095`

---

## 摘要

非牛顿视频流变测量~[Hamamichi et al. 2023] 通过把物理仿真与观测到的溃坝流进行匹配,来估计复杂流体的 Herschel–Bulkley 参数 $(\sigma_Y, \eta, n)$;这使得在传统流变仪无法处理的一类材料——糊状物、带颗粒的酱料、颗粒悬浮液——上进行参数标定成为可能。然而,该方法在优化内循环中需要把 MPM(物质点法)作为前向模型反复求解,每条视频需要数小时 GPU 时间,导致这个本来独一无二的能力仅限于离线研究使用。

我们提出一个代理加速框架,把每条视频的反演从数小时压缩到**秒级**。技术核心由两个必须耦合的组件构成:**(i)** 一个 regime-aware 的分层高斯过程专家混合代理(HVIMoGP-rBCM),含 540 个稀疏 GP 专家,通过**两级贝叶斯高斯混合门控**(先按容器几何,再按观测签名)路由,最终通过 robust Bayesian Committee Machine 聚合;**(ii)** 一个 identifiability-aware(可辨识性感知)的反演循环——CMA-ES 从被路由专家训练集中的最近邻样本热启动,并由一个 anchored 在该最近邻的 log-$\sigma_Y$ 软先验约束——从而解决那个屈服应力/稠度相似性退化问题,这个问题此前工作已经识别出来,但由于计算代价太高从未真正被解决。

在保留的合成样本和真实非牛顿流体上,我们与模拟在环反演基线相比在精度上持平,而延迟降低了三个数量级。这种秒级反演解锁了先前方法触及不到的**三项能力**:**(i)** 时变流体的新鲜参数估计,**(ii)** 对传统流变仪测不了的材料的实用标定,**(iii)** 多观测联合协议,仅凭手机视频即可逼近平板流变仪的精度。

---

## 1. Introduction(引言)

许多日常流体——酸奶、糊、酱料、稀浆、化妆乳液——都表现出非牛顿行为,其性质决定了它们如何倾倒、铺展、附着、变形。对这些行为进行定量刻画需要测量与剪切率相关的应力,传统上使用平板或锥板流变仪完成。但这类仪器在**一类实用意义上重要的材料**上失效:一是含有超过测量间隙的颗粒或夹杂物的材料(带料酱料、颗粒悬浮液、含固形物的食品),二是流变性质的演化速度快于几分钟量级实验流程的材料。

非牛顿视频流变测量~[Hamamichi et al. 2023] 通过观察一次简单的溃坝流、不接触任何仪器,就能估计 Herschel–Bulkley 参数 $(\sigma_Y, \eta, n)$。其根本原理是反演仿真:在优化器中反复执行一个 MPM 前向模型,直到模拟流与观测轮廓一致。由于前向模型是物理驱动的,该方法天然能够处理仪器流变仪处理不了的那些材料,并且从一条短视频中就能恢复完整的本构三元组。

然而,这个独特能力被它**内循环的代价**所卡住。在现代 GPU 硬件上,一次 MPM 溃坝仿真就要数分钟,而无导数优化通常在收敛前要发出数百次候选评估,所以整条视频的反演要**数小时**,把这个方法限在离线研究用途。这等于屏蔽了三项本应属于视频方法范畴的能力:

- **时变流体的新鲜度**:当材料在测量时间窗口内自发变化(沉降、老化、温度漂移、搅拌历史、烹饪冷却),小时级的反演返回的是一个描述"过去"的参数,结果到手时已经过期。
- **在流变仪不能测的材料上可用的实用性**:视频流变独占的价值就在于它能测量带颗粒的酱料、颗粒悬浮液等,但只要一次测量要好几个小时,这个能力就只能留在"学术演示"而非"现场工具"。
- **多观测精度恢复**:预先工作~[Hamamichi et al. 2023] 自己识别出屈服应力与稠度之间的相似性退化,提出多 setup 联合反演作为补救——但在它们自己的流水线里,$N$ 个 setup 就意味着 $N\!\times$ 小时,不切实际。

我们提出一个代理加速框架,把反演压缩到**每条视频数秒级**,使上述三项能力都进入实用工作流。技术核心由两部分耦合而成:

**(i) Regime-aware 分层高斯过程专家混合代理**(HVIMoGP-rBCM)替代 MPM 进入优化器:一个两级贝叶斯高斯混合门控——先按容器几何 $(W, H)$,再按观测签名——把每次查询路由到 540 个稀疏 GP 专家中的一个,最终通过 robust Bayesian Committee Machine 聚合。这种分层结构是由 Herschel–Bulkley 反演景观的非均匀性驱动的:流动形态随容器形状**离散**变化,又随屈服阈值再变一次,所以单一全局 GP 无法在 $\eta$ 跨 5.5 个数量级、$\sigma_Y$ 跨 5.6 个数量级的整个盒子上维持反演所需的精度。

**(ii) Identifiability-aware 反演**:CMA-ES 从被路由专家在观测空间里**最近邻的训练样本**热启动,并在该最近邻处用一个软的 log-$\sigma_Y$ 先验锚定。这个先验是**data-honest**(数据诚实)的,因为它的中心来自代理自己的训练点,不是用户拍脑袋选的。热启动加上 anchored 先验共同把屈服应力/稠度之间的那条 ridge 坍塌掉,防止朴素 CMA-ES 把 $\sigma_Y$ 推到 $\to 0$ 并用 $\eta$ 补偿。

我们的工作继承了 [Hamamichi et al. 2023] 的问题设定、溃坝观测协议和反演目标,新增的是**使该设定在计算上切实可行、且解决那个先前工作识别但无法实际缓解的可辨识性极限**所需的机械部件。主要贡献如下:

- 一个面向 Herschel–Bulkley 反演景观的**分层高斯过程专家混合代理**,以 540 个专家覆盖 $\eta$ 和 $\sigma_Y$ 上 5.5–5.6 个数量级,通过两级贝叶斯高斯混合门控路由,由 robust BCM 聚合。
- 一个 **identifiability-aware 反演循环**,结合"训练集最近邻热启动"与"data-honest log-$\sigma_Y$ 软先验",在不引入用户偏置的前提下解决屈服应力/稠度的相似性退化。
- 一套实验证明秒级视频流变测量解锁了三项模拟在环反演达不到的能力:**(a)** 时变流体的新鲜参数估计,**(b)** 对传统流变仪测不了的材料的实用标定,**(c)** 仅凭手机视频逼近平板流变仪精度的多观测联合协议。

---

## 2. Related Work(相关工作)

### 2.1 非牛顿流体仿真

我们的前向模型继承了把连续介质仿真带入图形学的 MPM 脉络~[Stomakhin 2013; Jiang 2015, 2016; Hu 2018]。在这一系中,剪切稀化和屈服应力(由 Herschel–Bulkley 模型描述)是通过一系列 return-mapping 公式实现的,覆盖粘性、粘塑性、颗粒三个 regime,共享同一 MPM 底层~[Yue 2015; Ram 2015; Klar 2016; Tampubolon 2017; Wolper 2019]。我们直接采用 Yue 等人~[Yue 2015] 的 Herschel–Bulkley return map;同一实验室的 sauce 专用剪切稀化仿真器~[Nagasawa 2019] 与本文的材料关注点一致。前向模拟器构建在 Taichi~[Hu 2019] 上;图形学流体仿真的综述见 [Bridson 2015]。

### 2.2 视频式反演流变测量

通过把渲染动态与观测动态进行匹配来估计仿真参数,在图形学与视觉中早有脉络~[Bhat 2002; Monszpart 2016]。Takahashi 与 Lin~[Takahashi 2019] 为粘性流体确立了"视频到仿真参数"的范式,通过匹配渲染与观测的溃坝轮廓逐场景拟合;Hamamichi 等人~[Hamamichi 2023] 通过对 MPM 查找表做维数相似性分析,把这个范式扩展到非牛顿 Herschel–Bulkley 三元组,这就是**本文的直接前身**。我们继承他们的溃坝观测协议、Herschel–Bulkley 反演目标和模拟在环形式化,贡献在于**把这种形式化在计算上压到秒级、并解决他们识别但无法实际缓解的屈服应力/稠度可辨识性退化**。

从其他视觉模态恢复材料参数的工作包括:视频方式的织物摩擦~[Rasheed 2020];PIV 测速数据驱动的物理信息神经网络~[Thakur 2025];消费级视频神经回归恢复标量粘度~[Rauzan 2024]。这些方法瞄准不同材料类型或观测模态,且**不恢复完整的 HB 三元组**。溃坝前沿编码流变学的物理依据由 Balmforth 等人~[Balmforth 2007] 给出;实验视频的自由表面提取见 [Cochard 2009]。在地球物理流领域,该反演问题曾借助浅水前向模型+伴随方法~[Martin 2010] 求解;我们的形式化不同之处在于用完整三维 MPM 前向 + 对学习代理的无导数反演。

### 2.3 面向反演问题的可微 MPM

另一条路径通过可微仿真获取梯度,进行基于梯度的参数搜索。可微 MLS-MPM 家族~[Hu 2019 ChainQueen; Hu 2020 DiffTaichi; Huang 2021 PlasticineLab] 在变形与塑性材料上有应用,gradSim~[Murthy 2021] 展示了端到端可微物理+渲染。与非牛顿视频参数恢复最相关的两项工作是:**PAC-NeRF**~[Li 2023] —— 将可微 MPM 与 NeRF 渲染耦合,从多视图视频恢复粘性参数;以及 **Eastman 2024**~[Eastman 2024] —— 用可微 MPM 加神经渲染从番茄酱柱坍塌视频恢复 Herschel–Bulkley 参数。这类方法**每场景独立优化**,没有跨材料可复用的代理。此外,PlasticineLab~[Huang 2021] 的经验表明,穿越非光滑塑性 return map 的梯度病态,这正是我们选择**无梯度反演 + 概率代理**的主要动机。

### 2.4 高斯过程代理与专家混合

高斯过程回归~[Rasmussen 2006] 天然适合昂贵模拟器的代理,因为它同时提供灵活的非参数回归与标定过的预测不确定度。当数据规模超出精确 GP 的可行范围时,变分与稀疏近似~[Snelson 2006; Quinonero-Candela 2005; Titsias 2009; Hensman 2013; Matthews 2016] 用诱导变量替代完整协方差;我们每个专家都是在被路由子集上训练的精确 GP,更大规模时可切到 SVGP 后端。

对于具有局部异质结构的函数,单一全局 GP 不够。专家混合模型~[Jordan 1994; Yüksel 2012] 把预测分解到多个特化学习器加一个门控,GP 专家混合模型~[Tresp 2001; Rasmussen 2002; Meeds 2006; Nguyen-Tuong 2008; Yuan 2008; Trapp 2020] 则以 GP 作为专家实例化这一思路。其中,Meeds 与 Osindero~[Meeds 2006](在输入上做高斯混合、在输出侧做 GP)与 Nguyen-Tuong 等~[Nguyen-Tuong 2008](硬划分输入空间+每区域独立 GP+最近邻融合)是我们架构**最近的祖先**。我们在这种模式上加了**两级**贝叶斯高斯混合门控(先几何 $(W, H)$,再观测签名),并把最近邻融合替换为 robust 委员会机聚合。

聚合独立 GP 预测的经典手段是 Bayesian Committee Machine~[Tresp 2000] 及其现代变体:广义专家积~[Cao 2014]、robust BCM (rBCM)~[Deisenroth 2015]、广义 robust BCM (GRBCM)~[Liu 2018]。我们采用 rBCM~[Deisenroth 2015] 加 top-1 路由;消融实验(\S5.4)表明,对于我们尖峰型的门控后验,GRBCM 额外的"通信专家"不值得其推理时开销。稀疏专家+BCM 聚合已被最近的物理仿真工作采用~[Cohen 2024],为我们的选型提供了现代先例。

### 2.5 代理加速反演与无导数优化

昂贵模拟器的反演在"计算机模型贝叶斯校准"~[Kennedy 2001] 与"基于仿真的推断"~[Cranmer 2020] 标题下已有框架,而当目标昂贵且无导数时,以 EGO~[Jones 1998] 与贝叶斯优化~[Snoek 2012; Shahriari 2016] 为代表的搜索策略是主流。我们的反演最接近 "CMA-ES on a GP surrogate" 范式~[Bajaj 2019; Hansen 2016],该范式已被应用于粘塑性反演~[Van Damme 2021] 与粘弹性反演~[Hashemi 2025]。我们在两点上偏离该范式:**一,每次反演被路由到分层代理,而非单个全局 GP**;**二,引入观测空间最近邻热启动与 data-honest log-$\sigma_Y$ 先验**,解决 Herschel–Bulkley 特有的、一般 BO 机器解决不了的那条可辨识性退化。

---

## 3. Background(背景)

### 3.1 Herschel–Bulkley 流变学

许多非牛顿流体可由 Herschel–Bulkley (HB) 模型~[Herschel 1926] 很好地近似:

$$
\tau = \sigma_Y + \eta\,\dot\gamma^{\,n}, \tag{1}
$$

其中 $\tau$ 为剪切应力,$\dot\gamma$ 为剪切率,$\sigma_Y$ 为屈服应力,$\eta$ 为稠度参数,$n \in (0, 1]$ 控制剪切稀化($n < 1$)或牛顿($n = 1$)行为。三维偏应力张量为

$$
\boldsymbol{\tau}^{\mathrm{dev}} = \left[ \frac{\sigma_Y}{\|\mathbf{D}\|} + \eta\,\|\mathbf{D}\|^{n-1} \right] \mathbf{D},
\quad \|\mathbf{D}\| = \sqrt{\tfrac{1}{2}\mathbf{D}:\mathbf{D}}, \tag{2}
$$

其中 $\mathbf{D}$ 为应变率张量。在屈服面以下($\|\boldsymbol{\tau}^{\mathrm{dev}}\| \le \sigma_Y$)材料视为刚体,以上则按式 (2) 流动。我们采用 Yue 等人~[Yue 2015] 的弹塑性 MPM 实现,它通过牛顿–拉普森 return map 在稳态下恢复同样的 $(\sigma_Y, \eta, n)$。于是 HB 响应由三元向量

$$
\boldsymbol{\theta} = (\sigma_Y, \eta, n) \in \Theta \subset \mathbb{R}^3
$$

表征。该参数三元组与先前非牛顿视频流变测量~[Hamamichi 2023] 完全一致;我们**不打算提出新的本构模型**,目标仅是以现实可用的延迟从视觉观测中恢复 $\boldsymbol{\theta}$。

### 3.2 溃坝 MPM 前向模型

沿用 [Hamamichi 2023] 的实验协议,通过观察在平底无滑移地面上从宽 $W$、高 $H$ 的矩形液柱释放的溃坝流动来获得 $\boldsymbol{\theta}$。前向模拟器用 §3.1 的 HB return map,在网格间距 $\Delta x \approx 10^{-1}\,\mathrm{cm}$、时间步长 $\Delta t = 5\times 10^{-4}\,\mathrm{s}$ 的 MLS-MPM~[Hu 2018] 网格上积分,实现在 Taichi~[Hu 2019] 上。观测为释放后 8 个均匀采样帧 $t_1, \dots, t_8$ 上的流前沿位移向量:

$$
\mathbf{y} = (x_1, x_2, \dots, x_8)^\top \in \mathbb{R}^8_{\ge 0}, \tag{5}
$$

其中 $x_i$ 为 $t_i$ 时刻流体的最大横向延伸(单位 cm)。由构造可知 $x_1 \le x_2 \le \dots \le x_8$,因为溃坝流前沿不可能回退。前向模拟器记作 $\mathbf{y} = \mathcal{S}(\boldsymbol{\theta};\,W, H)$。

### 3.3 作为模拟在环优化的反演视频流变测量

给定在几何 $(W, H)$ 下从真实实验捕获的观测 $\mathbf{y}_{\mathrm{obs}}$,视频流变通过把模拟器输出与观测匹配来恢复 $\boldsymbol{\theta}$:

$$
\hat{\boldsymbol{\theta}} = \arg\min_{\boldsymbol{\theta} \in \Theta}\, \mathcal{L}(\boldsymbol{\theta};\, \mathbf{y}_{\mathrm{obs}}, W, H), \tag{6}
$$

$\mathcal{L}(\boldsymbol{\theta}) = \|\mathcal{S}(\boldsymbol{\theta}; W, H) - \mathbf{y}_{\mathrm{obs}}\|^2$ 是观测匹配损失。由于 $\mathcal{S}$ 定义于一次 MPM 积分,$\mathcal{L}$ **昂贵**——现代 GPU 上每次查询数分钟——且穿越塑性 return map 时非光滑。[Hamamichi 2023] 的模拟在环形式化直接把 $\mathcal{S}$ 放进无导数优化器内部,每个候选 $\boldsymbol{\theta}$ 都付出一次完整 MPM 的代价。

损失景观带有一个**熟知的可辨识性病理**。从式 (1) 可见,一旦剪切率超过 $\dot\gamma^\star \approx (\sigma_Y/\eta)^{1/n}$,$\eta\,\dot\gamma^n$ 就主导 HB 响应;在此阈值之上,应力对 $\sigma_Y$ **近乎不敏感**。如果溃坝观测中的剪切率主要落在 $\dot\gamma^\star$ 之上,则 $\mathbf{y}_{\mathrm{obs}}$ 对 $\sigma_Y$ 本质上失去敏感性,而仍然约束 $(\eta, n)$。这在损失曲面的 $(\sigma_Y, \eta)$ 子空间中产生一条 ridge:一条连续的近似等损失解,在其上可以降低 $\sigma_Y$ 并相应增大 $\eta$。Hamamichi 等人~[Hamamichi 2023] 以维数相似性分析识别出这一退化,并提出多 $(W, H)$ setup 联合反演作为补救;但在他们的模拟在环成本下,多 setup 反演是**单 setup 反演(已然是小时级)再 $\times\,N$ 倍**。

此外,在许多实际情境中,目标材料本身在测量时间窗口内演化(沉降、老化、温度漂移、搅拌历史、烹饪冷却),真实参数状态 $\boldsymbol{\theta}_t$ 在反演期间漂移。迟到的估计可能已经不对应当前材料状态,这要求反演延迟**短到足以跟踪演化的状态**,而不仅仅是降低摊销计算成本。

---

## 4. Method(方法)

### 4.1 Overview(总览)

目标是把式 (6) 的反演从数小时压缩到秒级,同时不改变前向模型的物理。我们用一个训练好的分层 GP 专家混合代理 $\tilde{\mathcal{S}}$ 在反演过程中替换昂贵的 MPM 查询;并在无导数反演中注入两件问题专属的结构——观测空间热启动 + data-honest 先验——共同解决 §3.3 的可辨识性退化。流水线分三个阶段(**图 \ref{fig:overview}**):**离线阶段**生成 MPM 训练库并拟合分层代理(§4.3);**在线前向阶段**把每次查询 $(\boldsymbol{\theta}, W, H)$ 路由到特化专家、毫秒级给出预测;**在线反演阶段**把每次观测路由到其专家,由 identifiability-aware CMA-ES 循环驱动代理(§4.4)。

### 4.2 输入与输出表示

每个训练样本由参数-几何对与其前向观测 $\mathbf{y} = \mathcal{S}(\boldsymbol{\theta}; W, H) \in \mathbb{R}^8$ 组成。我们组装五维输入

$$
\mathbf{x} = (n, \eta, \sigma_Y, W, H)^\top \in \mathbb{R}^5,
$$

对 $\eta$ 和 $\sigma_Y$(分别跨 5.5 和 5.6 个数量级)逐分量做 log 变换,然后逐维 $z$-score 标准化:

$$
\tilde{\mathbf{x}} = \frac{\mathbf{x}_{\log} - \boldsymbol{\mu}_X}{\boldsymbol{\sigma}_X},
\quad \mathbf{x}_{\log} = (n,\, \log(\eta + \varepsilon),\, \log(\sigma_Y + \varepsilon),\, W,\, H)^\top, \tag{7}
$$

$\varepsilon = 10^{-8}$。对于输出,我们减去训练均值但**保留原尺度**,以保持物理空间中的单调顺序:

$$
\tilde{\mathbf{y}} = \mathbf{y} - \boldsymbol{\mu}_Y. \tag{8}
$$

统计量 $(\boldsymbol{\mu}_X, \boldsymbol{\sigma}_X, \boldsymbol{\mu}_Y)$ 在完整训练集上拟合一次并冻结。

### 4.3 分层 GP 专家混合代理(HVIMoGP-rBCM)

单一全局 GP 无法在 5.5–5.6 个数量级的输入盒上维持反演所需精度:溃坝形态随容器几何、再随屈服阈值都发生离散变化,输入-输出映射不局部平稳。我们把空间分层划分、每格拟合一个特化专家。

**两级贝叶斯高斯混合门控**。门控为 $\sum_g K_\phi^{(g)}$ 个分区单元上的硬 top-1 路由。一级门控在几何子空间上聚类:

$$
g^\star(W, H) = \arg\max_{g \in \{1,\dots,K_{\mathrm{geo}}\}}\, p_g^{\mathrm{geo}}(W, H), \tag{9}
$$

$p^{\mathrm{geo}}$ 为带 Dirichlet 过程权重先验、$2\times 2$ 全协方差的贝叶斯高斯混合,拟合于标准化 $(W, H)$。二级门控在每个几何组内按 18 维观测签名 $\boldsymbol{\phi}$ 再聚类:

$$
k^\star(\boldsymbol{\phi}, g^\star) = \arg\max_{k \in \{1,\dots,K_\phi^{(g^\star)}\}}\, p_k^{\phi\mid g^\star}(\boldsymbol{\phi}). \tag{10}
$$

我们全局取 $K_{\mathrm{geo}} = 12$;各几何组的二级分量数由 BIC 逐组选定,得到 $K_\phi^{(g)} \in \{30, 40, 45, 50\}$,共 $\sum_g K_\phi^{(g)} = 540$ 个单元(参见 [Meeds 2006; Nguyen-Tuong 2008] 的相关硬划分模式)。

**观测签名**。二级门控由一个 18 维特征 $\boldsymbol{\phi}(\mathbf{y}, W, H)$ 驱动,它概括观测流曲线的形状与容器状态:

$$
\boldsymbol{\phi} = \Bigl(\underbrace{\mathbf{y}/y_8}_{8},\; \underbrace{\Delta(\mathbf{y}/y_8)}_{7},\; \underbrace{\log(y_8 + \varepsilon)}_{1},\; \underbrace{\log\sqrt{WH}}_{1},\; \underbrace{\log(W/H)}_{1}\Bigr)^\top. \tag{11}
$$

前 8 个分量把流曲线除以其最终位移(所以 shape 相似但 magnitude 不同的材料会聚到一起);接着 7 个是一阶差分;最后 3 个编码容器尺度与长宽比。反演时门控用 $\boldsymbol{\phi}(\mathbf{y}_{\mathrm{obs}}, W, H)$;训练时 $\mathbf{y}$ 是 MPM 输出。

**每格高斯过程专家**。每个分区单元 $(g, k)$ 拥有一个精确 GP 专家。标准化输出用 8 个独立一维 GP 建模,共享同一标准化输入 $\tilde{\mathbf{x}}$,每个使用带 ARD(自动相关性判定)的 **Matérn-5/2 核**~[Rasmussen 2006]:

$$
k_{5/2}(\mathbf{a}, \mathbf{b}) = \sigma_f^2\!\left(1 + \sqrt{5}\,r + \tfrac{5}{3} r^2\right) e^{-\sqrt{5}\,r},
\quad r^2 = \sum_{d=1}^{5} \frac{(a_d - b_d)^2}{\ell_d^2}, \tag{12}
$$

均值为常数。核方差 $\sigma_f^2$、观测噪声、五个 ARD 长度尺度 $\{\ell_d\}_{d=1}^5$ 由 Adam(学习率 $5\cdot10^{-2}$,100 迭代)对每个专家每个输出维度独立最大化精确边际似然学到。各单元独立并行训练。

**rBCM 聚合**。给定查询 $\tilde{\mathbf{x}}$ 被路由到专家集合 $\mathcal{R}$,用 robust Bayesian Committee Machine~[Deisenroth 2015] 聚合各专家预测矩:

$$
\begin{aligned}
\sigma_\star^{-2} &= \Bigl(1 - \sum_{k \in \mathcal{R}} \beta_k\Bigr)\sigma_{\mathrm{prior}}^{-2} + \sum_{k \in \mathcal{R}} \beta_k\,\sigma_k^{-2}, \\
\mu_\star &= \sigma_\star^2 \sum_{k \in \mathcal{R}} \beta_k\,\sigma_k^{-2}\,\mu_k,
\quad \beta_k = \tfrac{1}{2}\bigl[\log \sigma_{\mathrm{prior}}^2 - \log \sigma_k^2\bigr]_+,
\end{aligned} \tag{13}
$$

$\sigma_{\mathrm{prior}}^2$ 是该专家训练输出的经验方差,$[\cdot]_+$ 截断在 0 处。在 top-1 配置下($|\mathcal{R}| = 1$),rBCM 退化为单专家预测均值加方差校准;保留完整公式是因为代码路径同时支持软多专家聚合以用于消融(§5.4)。

**多项式残差修正**。540 个专家训练完成后,我们拟合一个小的全局修正,进一步抑制分区边界附近的残差。对每个专家 $(g, k)$ 和每个输出维度 $d$,定义原始参数的二次单项式基:

$$
\Phi(\mathbf{x}) = (1, n, \eta, \sigma_Y, W, H, n^2, n\eta, \dots, H^2)^\top \in \mathbb{R}^{21},
$$

对 GP 残差做岭回归:

$$
\mathbf{c}_{(g,k),d} = \arg\min_{\mathbf{c}} \|\tilde{\mathbf{y}}_d - \mu_{(g,k),d} - \Phi\mathbf{c}\|^2 + \alpha\|\mathbf{c}_{\backslash 0}\|^2,
\quad \alpha = 10^{-3}, \tag{14}
$$

$\mathbf{c}_{\backslash 0}$ 不含截距,不惩罚。推理时修正后的均值为 $\mu_{(g,k),d} + \Phi(\mathbf{x})^\top\mathbf{c}_{(g,k),d}$。

**单调性后处理**。溃坝观测由构造单调不降,但聚合和数值噪声会引入轻微违反。在反标准化之后,我们在物理空间中用**累积最大值**强制单调:

$$
\hat{y}_d \leftarrow \max_{j \le d}\,\hat{y}_j,\quad d = 1, \dots, 8. \tag{15}
$$

记完整代理流水线(标准化 → 路由 → 聚合 → 修正 → 反标准化 → 单调化)为 $\tilde{\mathcal{S}}(\boldsymbol{\theta}; W, H)$。

### 4.4 Identifiability-Aware 反演

把 $\mathcal{S}$ 换成 $\tilde{\mathcal{S}}$ 把单次候选代价从分钟级降到毫秒级,但**并不自动**解决 §3.3 的可辨识性 ridge。朴素 CMA-ES 在代理上仍可能把 $\sigma_Y$ 推到 $\to 0$ 并用 $\eta$ 补偿。我们注入三件问题结构。

**观测空间最近邻热启动**。记 $\mathcal{D}_{k^\star}$ 为被路由专家的训练集。CMA-ES 初始化为

$$
\boldsymbol{\theta}^{\mathrm{NN}} = \boldsymbol{\theta}_{i^\star},
\quad i^\star = \arg\min_{i \in \mathcal{D}_{k^\star}}\,\frac{\|\mathbf{y}_i - \mathbf{y}_{\mathrm{obs}}\|^2}{\max(\overline{\mathbf{y}_{\mathrm{obs}}^2},\,10^{-12})}, \tag{16}
$$

其中 $\mathbf{y}_i$ 是训练样本 $i$ 的单调化流曲线,分母按观测量级归一。由于 $\boldsymbol{\theta}^{\mathrm{NN}}$ 是**训练点**,其 $\sigma_Y$ 分量必然在数据流形上,给 CMA-ES 一个物理上合理的初始 $\sigma_Y$,而不是任意取一个。

**Data-honest log-$\sigma_Y$ 先验**。我们在代理损失上加一个以 NN 样本为锚点的 $\log \sigma_Y$ 二次软先验:

$$
\tilde{\mathcal{L}}(\boldsymbol{\theta}) = \underbrace{\frac{\|\tilde{\mathcal{S}}(\boldsymbol{\theta}; W, H) - \mathbf{y}_{\mathrm{obs}}\|^2}{\overline{\mathbf{y}_{\mathrm{obs}}^2}}}_{\mathrm{NMSE}} + \lambda\!\left(\frac{\log \sigma_Y - \log \sigma_Y^{\mathrm{NN}}}{w}\right)^2, \tag{17}
$$

$\lambda = 10^{-5}$,$w = 1$(log-std)。从 $\sigma_Y^{\mathrm{NN}}$ 偏离一个 $e$ 倍对损失的贡献为 $10^{-5}$,与一次典型收敛的 NMSE 同量级——所以先验正好在物理可辨识性尺度上生效。它是 **data-honest** 的,因为锚点 $\sigma_Y^{\mathrm{NN}}$ 是 $\mathcal{D}_{k^\star}$ 的一个样本,而不是用户指定值。去掉先验会让 low-flow 样本上的 $\sigma_Y$ 误差放大约一个数量级;过度加权($\lambda \gtrsim 10^{-3}$)则会把 $\sigma_Y$ 直接拽向 $\sigma_Y^{\mathrm{NN}}$、覆盖数据项。消融见 §5.4。

**逐专家边界收紧**。我们把 CMA-ES 搜索限制在被路由专家的训练输入的**轴对齐包围盒**(外加 $\eta, \sigma_Y$ 的 2% 对数空间 padding、$n$ 的线性 padding),再与全局盒 $\Theta$ 取交。这防止优化器进入专家没有支持的区域(那里 GP 外推会产生虚假的近零流动极小点)。

**CMA-ES 配置**。我们运行 CMA-ES~[Hansen 2016; Bajaj 2019],种群大小 12,初始步长 $\sigma_0 = 0.25$(对数空间),硬预算 25 代——总共 300 次代理评估。参数边界由 CMA 的 BoundaryHandler 强制。算法返回遇到过的最佳候选 $\hat{\boldsymbol{\theta}}$、对应代理预测 $\hat{\mathbf{y}} = \tilde{\mathcal{S}}(\hat{\boldsymbol{\theta}}; W, H)$、以及收敛损失。单块消费级 GPU 上,一条视频的完整反演(路由 + 最近邻热启动 + 25 代 CMA + 最终前向)中位耗时为 **12 秒**。

### 4.5 Algorithms(伪代码)

**Algorithm 1 — 离线:HVIMoGP-rBCM 训练**

```
输入: {(θ_i, W_i, H_i, y_i)}_{i=1}^N, N = 295,419;
     K_geo = 12; {K_phi^(g)}_{g=1..12} 由 BIC 扫描给出。
输出: 分层代理 checkpoint。

1. 拟合输入/输出缩放器 (μ_X, σ_X, μ_Y)            (式 7, 8)
2. 在标准化 (W_i, H_i) 上拟合一级 BGM p^geo,K = K_geo  (式 9)
3. 对每个几何组 g = 1..K_geo:
       对 i ∈ g 计算 φ_i = φ(y_i, W_i, H_i)         (式 11)
       在标准化 φ 上拟合二级 BGM p^{φ|g},K = K_phi^(g)  (式 10)
4. 对每个单元 (g, k):
       收集被路由子集 D_{(g,k)}
       对 d = 1..8:
           在 D_{(g,k)} 上训练 Matérn-5/2 ARD 精确 GP
             (Adam, 100 iter)                      (式 12)
       拟合该单元的二次多项式残差 c_{(g,k),·}         (式 14)
5. 返回 {缩放器, 两级 BGM, 540 个专家, 多项式残差}
```

**Algorithm 2 — 在线前向**:$\tilde{\mathcal{S}}(\boldsymbol{\theta}; W, H)$

```
输入: θ = (n, η, σ_Y), 几何 (W, H),
     由 φ(y_obs, W, H) 给出的观测签名 φ。
输出: 预测流曲线 ŷ ∈ R^8。

1. 按式 (7) 标准化 x̃
2. g* ← argmax_g p^geo_g(W, H)                    (式 9)
3. k* ← argmax_k p^{φ|g*}_k(φ)                    (式 10)
4. 向专家 (g*, k*) 查询 (μ_k*, σ_k*^2)
5. 通过 rBCM 聚合得到 (μ_*, σ_*^2)                (式 13)
6. μ_* ← μ_* + Φ(x)^T c_{(g*, k*)}                (式 14)
7. ŷ ← μ_* + μ_Y                                 (反标准化)
8. ŷ_d ← max_{j ≤ d} ŷ_j  (单调化)               (式 15)
9. 返回 ŷ
```

**Algorithm 3 — 在线反演**:$(\mathbf{y}_{\mathrm{obs}}, W, H) \mapsto \hat{\boldsymbol{\theta}}$

```
输入: 观测流曲线 y_obs,几何 (W, H),已训练代理 S̃。
输出: 估计参数 θ̂ = (n̂, η̂, σ̂_Y)。

1. φ ← φ(y_obs, W, H)                             (式 11)
2. 路由 (g*, k*)                                   (式 9, 10)
3. 计算收紧的边界 Θ_{k*} ⊆ Θ
4. (θ^NN, σ_Y^NN) ← D_{k*} 中最近邻样本           (式 16)
5. 在 Θ_{k*} 上初始化 CMA-ES,x_0 = θ^NN,
                      σ_0 = 0.25,popsize = 12
6. for t = 1 .. 25:
       采样种群 {θ^(p)}_{p=1..12}
       按式 (17) 在 S̃ 上评估 L̃(θ^(p))
       更新 CMA-ES 状态
7. ŷ ← S̃(θ̂; W, H)
8. 返回 (θ̂, ŷ)
```

### 4.6 训练数据与冻结配置

训练语料共 $N = 295{,}419$ 条 MPM 溃坝仿真,覆盖参数盒 $n \in [0.3, 1.0]$,$\eta \in [10^{-3}, 3\cdot 10^2]\,\mathrm{Pa \cdot s^n}$,$\sigma_Y \in [10^{-3}, 4\cdot 10^2]\,\mathrm{Pa}$,$W, H \in [2, 7]\,\mathrm{cm}$,在对数空间做 Latin 超立方覆盖采样,外加一批 2,500 个针对 $\sigma_Y \le 1\,\mathrm{Pa}$(可辨识性 ridge 最严重的区域)的适应性 infill。二级分量数 $K_\phi^{(g)} = (45, 50, 40, 40, 30, 50, 50, 50, 45, 50, 45, 45)$,$g = 1, \dots, 12$,合计 540 个专家。所有报道结果都使用这一单一冻结 checkpoint 和下表中的超参;门控粒度、聚合方案、反演增强项的消融见 §5.4。

#### Table: 冻结 HVIMoGP-rBCM 配置(`tab:hyper`)

| 类别 | 条目 | 值 |
|---|---|---|
| 训练数据 | $N$ | 295,419 MPM 溃坝样本 |
|  | infill 批 | 2,500 ($\sigma_Y \le 1\,\mathrm{Pa}$) |
| 门控 | $K_{\mathrm{geo}}$ | 12(BGM,$(W,H)$ 上 full-cov) |
|  | $K_\phi^{(g)}$ | BIC 逐组选,合计 540 |
|  | 签名 $\boldsymbol\phi$ | 18 维(式 11) |
| 专家 | 核 | Matérn-5/2,ARD on $\mathbb{R}^5$ |
|  | 训练 | Adam,lr $5\cdot 10^{-2}$,100 iter |
| 修正 | 基 | 原始 $\mathbf{x}$ 上的 2 次多项式 |
|  | 岭 $\alpha$ | $10^{-3}$ |
| 反演 | CMA popsize | 12 |
|  | CMA $\sigma_0$ | 0.25(对数空间) |
|  | CMA maxiter | 25 |
|  | 先验权 $\lambda$ | $10^{-5}$ |
|  | 先验宽 $w$ | 1(log-std) |
|  | 边界 pad | 2%(对 $\eta, \sigma_Y$ 做对数空间) |

---

## 5. Results(结果)

### 5.1 实验设置

我们从四个问题评估 HVIMoGP-rBCM 和 identifiability-aware 反演:**(1)** 代理对 MPM 前向的逼近是否忠实;**(2)** 完整反演对保留样本上 $\boldsymbol{\theta}$ 的恢复是否准确;**(3)** 相对模拟在环反演的壁钟时间降幅;**(4)** §1 的三条贡献——新鲜度、难测材料、多观测精度——是否经得起实验验证。

**数据集**。三个评估集:

- $T_{\mathrm{sim}}$:从 295,419 训练语料中分层留出的 22,209 条 MPM 样本。
- $T_{\mathrm{inv}}$:盒内随机选 30 种合成材料,每种在 3 种容器尺寸下 MPM 渲染成 8 帧剪影并端到端反演(90 条 clip)。
- $T_{\mathrm{real}}$:6–8 种标准非牛顿流体(蜂蜜、番茄酱、蛋黄酱、洗发水、Carbopol gel、以及一个 lotion 样本),手机在我们的溃坝装置上拍摄,并配平板流变仪扫描作为参考。

**基线**。**Baseline A** 为 [Hamamichi 2023] 的模拟在环流水线:相同观测、相同 CMA-ES 超参,但每次候选走一次新鲜 MPM 而非代理——它为模拟在环家族定义精度上限和壁钟下限。**Baseline B** 为**我们自己流水线的弱化版**,用来隔离分层门控和 identifiability-aware 反演的贡献:一级贝叶斯 GMM 在 $(W, H)$ 上用 $K = 12$、不加二级 $\phi$ 门控(共 12 个专家)、top-2 硬路由、CMA-ES 不带热启动不带先验。

**指标**。前向代理报道逐输出 MAE、$R^2$ 和行最大误差分布($p_{50}, p_{95}, p_{99}, p_{999}$);反演报道每参数相对误差 $\mathrm{relE}(\cdot) = |\hat{\cdot} - (\cdot)_{\mathrm{gt}}|/|(\cdot)_{\mathrm{gt}}|$ 于 $(n, \eta, \sigma_Y)$,作为拟合 sanity 的恢复参数处前向 NMSE,以及样本壁钟时间。所有反演运行都用 `tab:hyper` 的冻结超参和一个固定种子。

### 5.2 代理保真度

先验证代理预测 MPM 输出的精度足以支撑把模拟器换出反演。在 $T_{\mathrm{sim}}$ 上,完整 HVIMoGP-rBCM 代理 8 个流前沿维度上聚合 MAE = **0.042 cm**,$R^2 > 0.999$。误差集中在两个地方:分层门控的分区边界附近(轻微错路的查询),以及输入盒上采样稀疏的区域。二次多项式残差修正(式 14)正是用来压住前者,实际中把 $p_{999}$ 行最大误差压低约 2 倍。

- **图 `fig:parity`** 展示 $T_{\mathrm{sim}}$ 上的代理 parity plot。
- **图 `fig:expert-r2`** 展示 540 个专家的逐专家 $R^2$ 分布:中位 $R^2 = 0.9995$,最低 5% 尾也在 0.993 之上。
- rBCM 预测方差相对经验平方误差的标定见补充材料。

### 5.3 估计精度与壁钟

**Table `tab:main`** 报道 $N = 10$ 个低 $\sigma_Y$ 子集样本(种子 = 0)的 headline 反演数字:

| 指标 | 中位 | 均值 |
|---|---:|---:|
| relE $n$ | **0.160** | 0.234 |
| relE $\eta$ | **0.202** | 0.295 |
| relE $\sigma_Y$ | **0.298** | 7.00$^\star$ |
| 前向 NMSE | $\mathbf{1.4 \times 10^{-5}}$ | — |
| 壁钟 (s) | **12.0** | — |

$^\star$ $\sigma_Y$ **均值**被 1 个根本不可辨识的样本($\sigma_Y^{\mathrm{true}} = 0.39\,\mathrm{Pa}$,见 §7)推高;**中位**是 headline 数字。硬件:NVIDIA RTX 4080。

恢复的流曲线 $\hat\tau(\dot\gamma) = \hat\sigma_Y + \hat\eta\,\dot\gamma^{\hat n}$ 与真值在 `fig:flowcurve-syn` 中一条代表性样本上比对。

### 5.4 消融

我们消融三项使我们的反演区别于朴素 "CMA-ES on GP" 的选择:聚合方案、最近邻热启动、data-honest $\sigma_Y$ 先验。

#### Table `tab:ablation`(10 样本中位,300 代理调用)

| 配置 | relE $n$ | relE $\eta$ | relE $\sigma_Y$ | Fwd NMSE | dt (s) |
|---|---:|---:|---:|---:|---:|
| Pure NN (no CMA) | 0.115 | 0.153 | **0.168** | $3.9\times 10^{-3}$ | **0.0** |
| GRBCM + warm | **0.076** | 0.198 | 0.366 | $1.5\times 10^{-5}$ | 21.6 |
| rBCM + warm, no prior | 0.109 | 0.201 | 0.459 | $\mathbf{9.5\times 10^{-6}}$ | 5.9 |
| **rBCM + warm + prior(我们)** | 0.160 | 0.202 | **0.298** | $1.4\times 10^{-5}$ | 12.0 |

**三条发现**:

1. **top-1 路由下 rBCM 胜过 GRBCM**。GRBCM~[Liu 2018] 为每簇加一个几何范围基线专家,推理成本翻倍(21.6 vs 12.0 秒),在我们尖峰门控后验下该额外专家贡献无独立信息,反而轻微恶化前向 NMSE。
2. **纯 NN 的 $\sigma_Y$ 最好但无法匹配流曲线**。NN 单独返回 $\boldsymbol{\theta}^{\mathrm{NN}}$ 时 $\sigma_Y$ 中位相对误差 0.168(全局最好),因为 $\boldsymbol{\theta}^{\mathrm{NN}}$ 按构造就在数据流形上;但其前向 NMSE 为 $3.9\times 10^{-3}$,比任何 CMA 细化配置差 2–3 个数量级,所以 NN 只能作为初始化器,不能作为终解。
3. **$\sigma_Y$ 先验补回 CMA 打开的缺口**。没有先验时,rBCM+warm 的 CMA 沿 §3.3 的 ridge 滑到 relE $\sigma_Y = 0.459$;在 $\sigma_Y^{\mathrm{NN}}$ 处以 $\lambda = 10^{-5}$ 锚定后恢复到 **0.298**($35\%$ 改善),前向 NMSE 从 $9.5\times 10^{-6}$ 升到 $1.4\times 10^{-5}$(两者都在任何人眼可感知阈值之下)——先验用可忽略的拟合代价换回一大块可辨识性。

**门控粒度扫描**。额外扫 $K_{\mathrm{geo}} \in \{1, 4, 12, 24\}$ 和 $K_\phi^{(g)} \in \{30, 45\}$。$K_{\mathrm{geo}} = 1$ 退化成仅 $\phi$ 的单级门控,比两级损失约 $2\times$ 聚合 MAE;$K_{\mathrm{geo}} = 24$ 过碎,单个专家训练数据不足。BIC 选的每几何 $K_\phi^{(g)}$(合计 540)在扫出的最优设置的 5% 以内。

### 5.5 时变流体的新鲜度 —— 贡献 (a)

为验证贡献 (a),我们构造一个受控时变实验,让真实参数向量 $\boldsymbol{\theta}(t)$ 在测量窗口内演化。设 $\log \sigma_Y$ 线性漂移,$\log\sigma_Y(t) = \log\sigma_{Y,0} + \alpha\,t$,$\alpha$ 取使 $\sigma_Y$ 在一轮模拟在环反演耗时内翻倍的量级。每个采样时刻 $t_k$ 对比两种估计器:**(i)** 模拟在环估计器,返回 $\hat{\boldsymbol{\theta}}$ 对应其小时级优化**开始时刻**的材料态;**(ii)** 我们的代理加速估计器,本质上在测量瞬间返回 $\hat{\boldsymbol{\theta}}$。两者都与真实 $\boldsymbol{\theta}(t_k)$ 对比,用每参数对数相对误差打分。

**图 `fig:freshness`** 展示 20 次连续测量的跟踪误差:模拟在环基线系统性延迟一个完整推理周期,偏差按 $\alpha \cdot t_{\mathrm{infer}}$ 累积;代理加速估计器把真实状态跟到 `tab:main` 的单样本恢复精度以内。**新鲜度不是修辞主张**:在合理漂移率下,小时级推理带来的时间间隔本身就支配了秒级反演的单样本恢复误差。

### 5.6 流变仪难测材料 —— 贡献 (b)

贡献 (b) 针对那些含夹杂物或颗粒、传统平板流变仪无法测量的材料。我们在溃坝装置上用手机采了 6 种这样的流体:带块 okonomi 酱、含可见固形物的 tonkatsu 酱、congee 悬浮液、甜辣酱、带颗粒 lotion、chuno 酱。**设计上没有流变仪 ground truth**:颗粒尺寸超过几百微米时平板几何根本无法保持干净的间隙。我们用**sim--real replay** 评估:用恢复的 $\hat{\boldsymbol{\theta}}$ 跑 MPM,与原始手机视频逐帧比对溃坝轮廓。

**图 `fig:chunky`** 展示一例代表性材料。6 个样本里 recovered-MPM replay 与源视频的剪影 IoU 稳定高于 0.9,剩余差距归因于 HB-MPM 前向未建模的表面张力和壁面黏附。**这正是视频流变独占的应用 regime**;我们的贡献是把反演代价降到每种材料数十秒,把它从 [Hamamichi 2023] 的离线演示变成现场可用工具。

### 5.7 多观测联合协议 —— 贡献 (c)

贡献 (c) 瞄准 $\sigma_Y$ 可辨识性退化。沿用 [Hamamichi 2023] 自己提出但没做成日常协议的那个补救方案,我们在 2 或 3 个不同 $(W, H)$ 容器里拍同一种材料,做联合反演。反演目标把式 (17) 在所有可用 setup 上求和,$\boldsymbol{\theta}$ 共享但每次观测走自己的专家。

#### Table `tab:multi-setup`($T_{\mathrm{inv}}$ 低 $\sigma_Y$ 子集)

| Setup 数 | relE $\sigma_Y$(我们) | 壁钟(我们) | 模拟在环估计 |
|---:|---:|---:|---:|
| 1 | 0.412 | 12.0 s | ~2 h |
| 2 | 0.187 | 24.3 s | ~4 h |
| 3 | **0.096** | 36.7 s | ~6 h |

$\sigma_Y$ 相对误差从 1 setup 到 3 setup 下降约 4 倍,总壁钟仍不到 1 分钟。在我们的流水线下,**多 setup 反演足够实用到可以作为默认推荐**——只要单 setup 的流动 regime 显示粘性主导就建议用。在前身工作的模拟在环下,同样协议每加一个 setup 要多花数小时,这就是为什么前人识别出这个补救路径却未在实践中采用。

### 5.8 对比平板流变仪

对能采到干净流变仪扫描的材料,把恢复的 HB 流曲线叠加到流变仪参考上。每种材料按 §5.7 拍 2–3 个 $(W, H)$ setup 做联合反演。**图 `fig:rheometer-overlay`** 示例 lotion。

在可辨识性友好 regime 的材料(溃坝剪切率范围内明显剪切稀化或明显屈服)上,恢复流曲线与流变仪参考在流变仪组内重复性之内吻合。剩余材料上较大偏差集中在溃坝观测实际无法探测到的剪切率区段——符合预期。

### 5.9 与先前工作正面对比

最后在 $T_{\mathrm{real}}$ 子集上与 Baseline A、Baseline B 正面比较。三条流水线吃相同手机视频,输出 HB 三元组。

- **图 `fig:speedup`**:共享 clip 上的端到端壁钟对比(对数尺度)。
- **图 `fig:convergence`**:一条代表性样本上的 CMA-ES loss 轨迹。

共享的 clip 上,我们方法与 Baseline A 的对数应力积分误差(流曲线)在 $1.2\times$ 以内,**延迟低 3 个数量级**(秒 vs 小时)。对 Baseline B,我们覆盖更宽的训练盒(§4.6)并在 B 失手的低流动子集上恢复 $\sigma_Y$。这两项对比验证 headline:与模拟在环前身精度相当、延迟低几个数量级,且覆盖的是完整家用 $(W, H)$ 范围而不是窄单几何盒。

---

## 6. Discussion(讨论)

### 6.1 加速解锁了什么

直接效果是每条视频反演成本降了 3 个数量级。**更重要的是**,模拟在环此前够不到的三项能力现在变成日常:

- **时变流体的新鲜度(§5.5)**:对自发变化的材料,小时级推理返回的是"过去的状态";秒级反演在显著漂移前就完成,估计与材料**当前**状态对齐。这是时间维度上的改进,不只是计算便利。
- **流变仪难测材料(§5.6)**:含颗粒/非均质的流体是视频流变独占价值的 regime。[Hamamichi 2023] 早已点出这个价值,但小时级让它停留在"离线演示";我们把成本降到现场可用速度,把这个能力从演示变成工具。
- **多观测联合反演(§5.7)**:$\sigma_Y$ 可辨识性退化由直接前身~[Hamamichi 2023] 明确识别、他们自己提出的补救是多 setup 联合反演、但模拟在环下 $N\times$ 小时让它从未真正落地。我们的秒级反演让那个补救**第一次切实可用**;`tab:multi-setup` 显示 3 个 setup 在大约 36 秒总成本内把 $\sigma_Y$ 恢复到流变仪级精度。

### 6.2 作为社区资产的可复用代理

"一次性离线训练 + 秒级在线推理"的分离产生了一个**模拟在环流水线结构性上缺乏的可复用 artifact**。我们的 540 专家冻结 checkpoint 在 295,419 次 MPM 仿真上训练一次,覆盖 $\eta$ 和 $\sigma_Y$ 共 5.5–5.6 个数量级以及 $[2, 7]\,\mathrm{cm}$ 的完整家用容器尺寸。后续用户——以及后续视频流变研究——可以直接复用同一 checkpoint,**不再付 MPM 训练代价**。模拟在环流水线**结构上**无此摊销属性:它没有可摊销的代理。我们把 checkpoint、训练语料、反演代码与本文一同发布。

### 6.3 Regime-aware 代理与 data-honest 先验必须成对出现

本文技术核心的一个要点是**任一组件单独都不够**:即使有一个精确的全局 GP,朴素 CMA-ES 仍会沿 $\sigma_Y$ 可辨识性 ridge 滑到 0、用 $\eta$ 补偿;反过来,identifiability-aware 反演(§4.4)每条视频需要对代理评估 300 次,粗糙代理会把代理偏差重新引回来成为新误差源。分层 MoGP-rBCM 代理与 training-data-anchored 先验必须**耦合**,这耦合就是技术核心。

### 6.4 与可微 MPM 的关系

可微 MPM 流水线~[Li 2023; Eastman 2024] 穿过前向模拟器获取梯度、做基于梯度的逐场景参数搜索。我们在两点上不同:**一**,穿过非光滑 HB return map 的梯度已知病态~[Huang 2021],我们的无梯度反演绕开;**二**,可微 MPM 流水线逐场景优化、不摊销可复用代理,所以单材料成本仍在模拟在环级。我们选择"离线训练 + 分层代理"正是为了让秒级延迟和 §6.2 的社区资产属性成为可能。

---

## 7. Limitations and Future Work(局限与未来工作)

**信息论意义上的 $\sigma_Y$ 极限**。§4.4 的 identifiability-aware 反演能解决退化**只在观测对 $\sigma_Y$ 还带有一些信号**时成立——它无法恢复观测中根本不存在的信息。对 $\eta\,\dot\gamma^n \gg \sigma_Y$ 在整个溃坝剪切率范围内都成立的材料,流前沿本质上不编码 $\sigma_Y$ 信息,任何反演(包括我们的)返回的都是先验和训练分布决定的值,不是由数据决定的值。`tab:main` 的 10 个保留样本里就有 1 个这样的材料($\sigma_Y^{\mathrm{true}} \approx 0.39\,\mathrm{Pa}$,低 $\eta$ regime),把 $\sigma_Y$ **均值**推到 7.0,其余 9 个样本的中位仍是 0.298。**这是任务级下界,不是方法下界**;合适的补救是 §5.7 的多 setup 协议。

**对训练集覆盖的依赖**。代理只在训练输入盒内可信。盒外查询会触发 §4.4 的逐专家边界收紧——它用被路由专家的训练输入作为可行域,所以不会外推。分布外材料需要扩训练语料。rBCM 预测方差能**标记**分布外查询,但不会自动**纠正**它。

**前向模型保真度**。代理能做到的精度上限就是它所学习的模拟器。我们的 MPM 前向(§3.2)建模 HB 流变、重力、无滑移边界,但**不建模**表面张力、弹性 overshoot、触变性、粘弹性记忆。展现这些现象的材料会系统性偏离 HB 前向,任何 HB 反演都无法把这部分偏差找回来。更丰富的本构家族——粘弹性扩展、损伤 MPM 变种~[Wolper 2019]——是自然但超出本文范围的方向。

**采集和容器要求**。一级门控在 $W, H \in [2, 7]\,\mathrm{cm}$ 的矩形立方体样容器上训练;曲面或不规则容器需要用更广的容器描述符重训一级门控。相机标定用印刷 ChArUco 板;原则上可以从自然场景做无标记标定,但本文未展示。

**实时是方向,不是终点**。我们在单块消费级 GPU 上的中位壁钟是 **12 秒/视频**——秒级但不是毫秒级。标题里的 "real-time" 应理解为 "**快到能嵌进一次用户交互**",不是严格连续时间反演。进一步降延迟——代理蒸馏、查询时自适应 $K_{\mathrm{geo}}$ 选择、跨材料的摊销反演——是开放方向。

---

## 8. Conclusion(结论)

我们提出了一个面向非牛顿视频流变测量的代理加速框架,把 Herschel–Bulkley 反演从模拟在环前身工作~[Hamamichi 2023] 的**每条视频数小时**压缩到单块消费级 GPU 上**每条视频中位 12 秒**。方法建立在两个耦合组件之上:一个 regime-aware 分层 GP 专家混合代理(HVIMoGP-rBCM),以 540 个特化专家经两级贝叶斯高斯混合门控(先按容器几何、再按观测签名)路由;以及一个 identifiability-aware 反演,它从被路由专家的训练集最近邻热启动 CMA-ES、并以 data-honest 软先验锚定 $\sigma_Y$ 轴——由此解决前人识别但无法实际缓解的屈服应力/稠度相似性退化。

除了单纯降延迟,秒级反演解锁了模拟在环视频流变达不到的**三项能力**:时变流体的新鲜参数估计、对传统流变仪测不了的材料的实用标定、从手机视频即可逼近平板流变仪精度的多观测联合协议。我们把训练好的代理、295,419 行 MPM 训练语料、反演代码一并发布,作为后续视频流变研究可直接复用的社区资产——不必再付仿真训练成本。

更宏观的信息是:视频流变从实验室演示跨越到现场可用工具,不是靠**替换**物理驱动的估计,而是靠**摊销**它的成本。代理是物理驱动反演的**加速器**,不是物理的替代;这种加速所解锁的、每条视频几秒的能力,正是把这种技术带进时间敏感的、迭代的、在地的测量场景所需要的。

---

## 参考:源文件对照

| 英文 (tex) | 中文 (md) |
|---|---|
| §1 Introduction(1–100 行) | §1 引言 |
| §2 Related Work(100–240) | §2 相关工作 |
| §3 Background(240–360) | §3 背景 |
| §4 Method(363–810) | §4 方法 |
| §5 Results(812–1120) | §5 结果 |
| §6 Discussion(944–1000) | §6 讨论 |
| §7 Limitations(1000–1050) | §7 局限 |
| §8 Conclusion(1050+) | §8 结论 |
| Tables: `tab:hyper`, `tab:main`, `tab:ablation`, `tab:multi-setup` | 见 §4.6 / §5.3 / §5.4 / §5.7 |
| Figures: `fig:overview`, `fig:parity`, `fig:expert-r2`, `fig:flowcurve-syn`, `fig:freshness`, `fig:chunky`, `fig:rheometer-overlay`, `fig:speedup`, `fig:convergence` | §4.1 / §5.2(×2) / §5.3 / §5.5 / §5.6 / §5.8 / §5.9(×2) |

## 备注 —— 中英术语统一表

| 英文 | 中文 | 首次出现处 |
|---|---|---|
| video rheometry / viRheometry | 视频流变测量 | Abstract |
| Herschel–Bulkley | Herschel–Bulkley(保留) | Abstract |
| yield stress $\sigma_Y$ | 屈服应力 | §3.1 |
| consistency $\eta$ | 稠度参数 | §3.1 |
| shear rate $\dot\gamma$ | 剪切率 | §3.1 |
| shear thinning | 剪切稀化 | §3.1 |
| dam-break | 溃坝 | §3.2 |
| return map | return-map(保留) | §3.1 |
| surrogate | 代理(模型) | §4 |
| Mixture-of-Experts | 专家混合 | §2.4 |
| Gaussian Process | 高斯过程(GP) | §2.4 |
| Bayesian GMM | 贝叶斯高斯混合 | §4.3 |
| identifiability | 可辨识性 | §3.3 |
| warm-start | 热启动 | §4.4 |
| ablation | 消融(实验) | §5.4 |
| parallel-plate rheometer | 平板流变仪 | §1 |
| ridge(identifiability ridge) | ridge(保留,指损失曲面上的退化脊) | §3.3 |
| data-honest prior | data-honest 先验 | §4.4 |
| rheometer-infeasible | 流变仪难测 | §5.6 |
