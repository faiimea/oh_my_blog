# 利用本征功率传输网络检测PCB异常

## 背景
### PDN
PDN全称为power delivery network，翻译为中文为电源分配网络。电源分配网络（PDN）就是将电源功率从电源输送给负载的实体路径。电流通过PDN从电源端流向负载端，再通过PDN，从负载端流回电源端。

实际生活中，任何金属都存在很低的阻抗和寄生电感。
阻抗很低，（R0=8mΩ），
寄生电感很低（L0=8pH），
C0和G0则依赖于正极导线和负极导线的距离。一般C0极小，G0极大。

负载大小和负载频率的变化，最终都会导致PDN传输的负载的电压不再与电源输出相等，一旦负载获得的电压低于负载的工作需求，就会导致负载发生不可预见性的故障。

### PDN阻抗曲线
PDN的阻抗分布（也称为Z参数）在频域被广泛用于评估其性能，其表示为对称矩阵。在具有电阻、电感和电容的电路里，对电路中的电流所起的阻碍作用叫做阻抗。阻抗常用Z表示，是一个复数，实部称为电阻，虚部称为电抗，其中电容在电路中对交流电所起的阻碍作用称为容抗 ,电感在电路中对交流电所起的阻碍作用称为感抗，电容和电感在电路中对交流电引起的阻碍作用总称为阻抗。

对于传统的PDN分析，只有自阻抗才有意义，因为它可以代表提供给芯片的电能质量。对于基于PDN的异常检测，我们还应该关注传输阻抗，因为板载电容器的行为就像屏障，将一个电压域分成多个子域。自阻抗可以精确地表征一个子域中的PDN，而传输阻抗可以以更高的噪声为代价跨多个子域进行检测。

（1）低频部分，这是由于PDN电路的电特性（特别是离散部件），和（2）高频部分，这主要是由于电源平面之间的PCB空腔形成的电磁共振（即板共振）。

曲线的两个部分分别有助于揭示PCB异常引起的电路电平变化和电路板谐振变化的影响。结合电路电平和电路板谐振信息，阻抗分布图捕捉到PCB设计中的微小变化，即使这些变化不会直接影响PDN电路本身的可操作性。在本文中，低频和高频信息一起用于检测PCB修改。


### VRM
电压调节模组（Voltage Regulator Module，简称：VRM）是为微处理器提供合适的供应电压的一项装置，它可以直接焊接在主板上，也可以用模组子卡的方式来安装，由于它可以变换调节供应电压，因此可以让同一片主板换装使用不同种供应电压的处理器。

在复杂的PCB设计中，芯片/组件对可靠运行有不同的功率分配要求，如电源电压水平、最大负载电流和电压噪声裕度。因此，PDN由VRM组成，VRM形成树状结构以创建多个电压域。

每个电压域都有自己的VRM来转换来自上节点电压域的电源，并将本地电源电压驱动到芯片。VRM还隔离两个电压域，因为其主要功能之一是防止一个域的电压波动传播到另一个域。断电时，VRM开路，从而断开两个域的连接。在分层VRM和芯片之间是板级无源配电网络，包括PCB电源线、电源平面和板上离散解耦电容器。

## abstract
印刷电路板（PCB）在现代电子系统和嵌入式设备中无处不在，这使得印刷电路板的整体性成为首要的安全问题。为了利用规模经济，当今的PCB设计和制造通常由全球各地的供应商完成，这使他们在细分的PCB供应链中面临许多安全漏洞。此外，PCB设计的日益复杂也为在PCB生命周期的每个阶段实施的无数偷偷摸摸的板级攻击留出了充足的空间，从而威胁到许多电子设备。

在本文中，我们提出了PDNPulse，一种基于电力传输网络（PDN）的PCB异常检测框架，可以识别广泛的板级恶意修改。PDNPulse利用了这样一个事实，即PDN的特性不可避免地受到PCB修改的影响。通过检测PDN阻抗曲线的变化，并使用基于Frechet距离的异常检测算法，PDN脉冲可以可靠且成功地识别整个系统中的恶意修改。使用PDNPulse，我们对七种商用现货PCB进行了广泛的实验，涵盖了不同的设计规模、不同的威胁模型和七种不同的异常类型。结果证实，PDNPU在攻击和防御之间产生了有效的安全不对称。

## Threat Model

攻击者可以通过插入特洛伊木马电路或执行假冒（包括低质量和回收的）替换来植入异常。我们表明，PDNPulse可以有效地检测大多数实际的板级攻击

* PCB特洛伊木马：特洛伊木马电路为攻击者创建后门，并可用于发起破坏安全保障的攻击。
* 芯片/组件假冒品：这些组件的安全性未经验证。因此，攻击者可以利用它们来发起攻击
* PCB赝品：此类假冒产品暴露了系统的漏洞，并增加了故障率。

**黄金模型**。请注意，PDNPulse从根本上确定测试板是否可信。在这项工作中，原装主板（即黄金型号）是由原始设备制造商（OEM）直接提供的，包括PCBitself及其板载电子组件。在构建黄金模型时，需要考虑PCB制造和组件变化引起的工艺变化。具体而言，部件变化可能是由于制造公差或采用多个BOM表引起的。如果PDN阻抗曲线的偏差超过了黄金模型的记录容差，我们可以合理地将被测电路板视为不可信（即恶意/伪造/可疑）。我们的方法并非旨在找出此类偏差的根本原因或恶意。

## Proposed detection methodology
在本节中，我们首先介绍所提出的PDNPulse框架的总体工作流程。然后，我们详细阐述了PDNPulse如何帮助检测不同的板级攻击，并讨论了测量PCB相对于攻击向量的阻抗分布时的挑战和注意事项。
### PDNPulse Framework
以循序渐进的方式。要构建新PCB设计的黄金模型，步骤1∼3应该完成一次，然后是步骤4以记录几个真正的PCB实例。为了验证新的PCB情况，用户只能执行步骤4和5

电压域选择 - 端口选择 - 实验设置 - Z参数测量 - 异常检测

### PDN Sensitivity

### Frechet Distance-based Anomaly Detection Algorithms
采用测量两条曲线之间相似性的Frechet距离[10]作为安全度量，以评估阻抗曲线之间的差异，并量化PDNPulse的唯一性和稳定性。

### Anomaly Detection



## PDNPU Analysis and results
在本节中，我们广泛分析了PDNPulse，以说明其捕获板级攻击的能力，包括PCB木马、芯片/组件仿冒和PCB仿冒。




## defending against adaptive attackers
在本节中，我们将讨论PDNPulse对试图故意绕过PDNPulle的不适应性攻击者的能力。首先探索检测灵敏度，以显示检测精心设计的特洛伊木马的性能。然后讨论了攻击者可以利用的其他更隐蔽的机制。总体而言，PDNPulse旨在在攻击和防御之间建立有效的安全对称。尽管PDNPulse不是PCB攻击的最终解决方案，但实施PDNPulse可显著减轻潜在威胁。最有动机的攻击者可以故意绕过PDNPulse，但代价可能是通过正交方法（例如，检查、功能测试和完整性检查）或要求显著先进的技术使其恶意植入更容易被检测。

attacker response：
我们探索可用的攻击向量，这些攻击向量要么通过避免连接到PDN来破坏PDNPulse，要么通过隐藏其对PDN阻抗分布的影响来试图绕过PDNPulse

* 对手可以避免连接PDN以绕过PDN脉冲。使用备用电源。对手可以使用自供电电路或从数据信号获取电力，以避免直接连接到PDN。攻击者需要采用显著先进的电池制造和集成技术才能成功


* 对PDN阻抗配置文件的隐藏影响：尽管特洛伊木马必须连接到PDN，但攻击者可能会试图减少对PDN配置文件的影响，以避免PDN脉冲。对手可能会学习PDNPulse的测量参数，例如测量的电压域和端口，然后有意将他们的特洛伊木马植入这些参数之外，以避免被检测到。必须彻底搜索解决方案以完全补偿PDN配置文件（所有Z参数）