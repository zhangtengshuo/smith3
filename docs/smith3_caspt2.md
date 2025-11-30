# SMITH3 与 CASPT2 自动代码生成简介

## 项目定位
SMITH3 是一个 **自动化生成自旋无关多参考电子相关程序的生成器**，核心任务是将给定的多体算符组合转换为可编译的 C++ 计算代码（如 CASPT2、MRCI 等方法）。【F:src/forest.h†L2-L78】【F:src/main.cc†L1-L40】

## 工作流程概述
1. 在 `main.cc` 中设置目标理论（默认 `CASPT2`），并列举激发算符、积分张量等，形成一系列残差或能量方程的算符序列。
2. `Forest` 类对这些算符树执行 Wick 展开、筛选唯一的中间张量（如 Gamma），并生成任务队列与头文件代码。
3. 生成的算法通过 `solve()`/`solve_deriv()` 接口调用：
   - `solve()` 根据所选理论拼接对应的主驱动（例如 CASPT2 或 MS-MRCI）。【F:src/forest.cc†L311-L318】
   - `solve_deriv()` 目前只对 CASPT2 提供导数驱动，会自动构造密度矩阵与 CI 微分的任务队列。【F:src/forest.cc†L320-L356】

伪代码（概念化）如下：
```pseudo
main():
    theory = "CASPT2"
    operators = build_residual_operators(theory)
    forest = Forest(trees_from(operators))
    forest.filter_gamma()
    code = forest.generate_code()
    write_files(code)
```

`solve_deriv()` 的核心逻辑：
```pseudo
solve_deriv():
    if theory != CASPT2: raise NotImplemented
    corrq = make_corrq()       # 生成关联范数队列
    correlated_norm = accumulate(corrq)

    dens2_queue = make_densityq()   # 二阶密度矩阵
    run_queue(dens2_queue)
    dens1_queue = make_density1q()  # 一阶密度矩阵
    run_queue(dens1_queue)
    dens1_corr_queue = make_density2q()  # 关联密度贡献
    run_queue(dens1_corr_queue)

    dec_queue = make_deciq()  # CI 波函数导数
    run_queue(dec_queue)
```

## CASPT2 能量与梯度的自动生成
- **能量部分**：`solve()` 在 CASPT2 下调用自动生成的主驱动 `caspt2_main_driver_()`，该驱动由符号展开结果直接拼装，负责残差构造与迭代求解。
- **梯度部分**：`solve_deriv()` 为 CASPT2 自动生成核梯度计算所需的全部中间量：
  - 计算关联范数 \(N = \langle \Psi_1 | \Psi_1 \rangle\)。
  - 构造一阶、二阶密度矩阵 \(\gamma^{(1)}, \Gamma^{(2)}\) 以及相关修正 \(\Delta\gamma^{(1)}\)。
  - 生成 CI 系数导数 \(\partial C / \partial R\)。【F:src/forest.cc†L320-L356】

这等价于基于 Lagrangian 的解析梯度公式：
\[
\frac{dE}{dR} = \frac{\partial \mathcal{L}}{\partial R} = \mathrm{Tr}[\gamma^{(1)} h^{(1)}_R] + \frac{1}{2}\mathrm{Tr}[\Gamma^{(2)} v_R] + \mathbf{z}^T \frac{\partial \mathbf{f}}{\partial R},
\]
其中 \(h^{(1)}_R, v_R\) 是一、二电子积分对核坐标 \(R\) 的导数，\(\mathbf{z}\) 为响应方程求得的拉格朗日乘子，\(\mathbf{f}=0\) 表示 CASPT2 残差。

## 是否支持其他方法的梯度
`solve_deriv()` 显式检查理论名称，当前只有 **CASPT2** 实现了导数（核梯度）。若选择 MRCI 或相对论变体，会抛出“未实现”逻辑错误。【F:src/forest.cc†L320-L356】

## 解析 Hessian（能量二阶导数）的生成思路
项目当前未直接提供 CASPT2/MSCASPT2 的解析 Hessian 自动生成，但可以沿用现有梯度生成框架，步骤可概念化为：

1. **扩展拉格朗日**：在梯度拉格朗日基础上加入一阶响应变量的微分项，建立二阶拉格朗日 \(\mathcal{L}^{(2)}\)。
2. **自动化二阶 Wick 展开**：对包含一次微分算符的残差与密度表达式再次调用 Wick 展开，生成新的任务树。
   ```pseudo
   build_second_order_tasks():
       terms = d/dR (first_order_terms)  # 对梯度公式的每一项再求导
       return wick_expand(terms)
   ```
3. **求解一次响应方程**：先调用已生成的梯度代码得到 \(\mathbf{z}\) 和密度矩阵，再构造并求解耦合双响应方程，获得轨道/CI 一次响应 \(\partial \mathbf{T}/\partial R\)。
4. **汇总 Hessian**：
   \[
   \frac{d^2E}{dR dS} = \frac{d}{dS} \left( \frac{dE}{dR} \right)
   = \mathrm{Tr}[\gamma^{(1)}_S h^{(1)}_R] + \mathrm{Tr}[\Gamma^{(2)}_S v_R] + \mathbf{z}_S^T \frac{\partial \mathbf{f}}{\partial R} + \mathbf{z}^T \frac{\partial^2 \mathbf{f}}{\partial R \partial S}
   \]
   其中带下标 \(S\) 的量由步骤 3 产生。
5. **代码生成实现要点**：
   - 复用 `Forest` 生成任务的接口，新增二阶密度、二阶 CI 导数的树定义。
   - 在 `solve_deriv()` 之外添加 `solve_hessian()`，流程类似但调用二阶队列生成函数。
   - 保持数据类型与并行队列 (`Queue`) 接口一致，方便与 BAGEL 主程序衔接。

以上步骤可以作为在 SMITH3 内部实现 CASPT2/MSCASPT2 解析 Hessian 自动生成的蓝图。
