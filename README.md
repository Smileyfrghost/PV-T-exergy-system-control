# PV-T-exergy-system-control
A code-based simulation of PV-T system and control system
complete with the assistant of gemini3.1 pro
# PVT System Simulation & Control Framework (v6.0)

## 1. Project Overview
This project provides a high-fidelity simulation and control framework for Photovoltaic/Thermal (PVT) systems. The core objective is to evaluate advanced flow control strategies that balance electrical efficiency and thermal grade (Domestic Hot Water - DHW).

**Important Note**: Version 6.0 implements a rigorous physical foundation based on established literature (Duffie & Beckman, Bahaidarah 2013). This package is a research-grade prototype and is not intended for safety-critical industrial deployment without further hardware validation and grid-stability analysis.

## 2. Physical & Thermodynamic Optimization Algorithms (Scientific Rigor)
The physical engine of the simulator and the optimization algorithms have been completely refactored in v6.0 to strictly adhere to thermodynamic principles and steady-state thermal modeling.

*   **Modified Hottel-Whillier-Bliss (HWB) Model**: The system implements an extended HWB analytical model to accommodate the electrical-thermal coupling effect. Electrical power generation is accurately treated as energy removal, scaling the effective overall loss coefficient ($U_L^*$) and absorbed solar radiation ($S^*$) dynamically.
*   **Net Exergy (Total Useful Work) Optimization**: The traditional multi-objective (thermal vs. electrical) optimization has been unified under Petela's Exergy theory (1964). The optimization algorithm evaluates the net exergy gain ($E_{net} = E_{thermal} + E_{electrical} - E_{pump}$). Thermal exergy is strictly derived using logarithmic mean temperature difference (LMTD), considering the reference dead-state boundary conditions as the time-varying ambient temperature ($T_a$).
*   **Convection & Radiation**: Implements the **Swinbank model** for sky temperature ($T_{sky} = 0.0552 \cdot T_a^{1.5}$) and linearized Stefan-Boltzmann radiative exchange. Internal flow heat transfer coefficient ($h_{fi}$) adheres to the **Dittus-Boelter** correlation, mapping mass flow directly to convective efficiency.
*   **Hydraulics & Pumping Power**: Implements a physical pressure drop model based on the Blasius friction factor to rigorously evaluate the parasitic pump power penalty, penalizing excessive mass flow strategies ($E_{pump} \propto \dot{m}^3$).

## 3. Microcontroller (MCU) Firmware Architecture
The embedded controller (`CORE_pvt_mcu_control.c`) has been rewritten to Automotive-grade/MISRA-C standards to ensure deterministic execution for STM32 hardware deployment.

*   **Absolute Reentrancy and Context Isolation**: All state variables have been abstracted into a `PVT_Controller_Context` structure. The firm exclusion of `static` global variables guarantees thread safety and supports Multi-tasking RTOS schedulers.
*   **Zero-RAM Large-Scale Look-Up Table (LUT)**: 
    *   To bypass the computational overhead of solving non-linear HWB equations onboard, the system utilizes a 61x31 high-resolution float table (representing optimal flow trajectories).
    *   By strictly declaring the arrays as `static const`, the entire dataset (>100 KB) is mapped into the Flash ROM, reducing SRAM consumption to nearly zero, avoiding stack-overflow risks in constrained MCUs.
*   **Bilinear Interpolation Engine**: The continuous feedforward mass flow target is obtained from the discrete LUT parameter space via two-dimensional Bilinear Interpolation, ensuring first-order continuity in control actuation without numerical ringing (Runge's phenomenon).
*   **Discrete PID with Gain Scheduling**:
    *   **Time-Domain Consistency**: Introduces discrete sampling time (`Ts`) terms to the integral and derivative equations to eliminate temporal aliasing between Python continuous simulations and MCU step executions.
    *   **Gain Scheduling**: The proportional ($K_P$) and integral ($K_I$) gains are dynamically scheduled as linear functions of the current target flow. Low flow conditions employ conservative gains to suppress overshoot, while high flow scenarios deploy aggressive gains for rapid transient response.
    *   **Conditional Integration Anti-Windup**: Eliminates conventional algorithmic jitter by employing a forward-integration freezing mechanism. The integrator pauses dynamically based on actuator saturation and directional error to neutralize limit-cycle oscillations.

## 4. Scientific Data Visualization and Plotting Framework
The system implements a dual-pipeline data visualization architecture dedicated to analytical robustness and publication readiness.

*   **Python/Matplotlib Fast Dashboards (`utils_matplotlib_plot_*.py`)**:
    *   `utils_matplotlib_plot_transient.py`: Visualizes time-domain state variables (Irradiance, $T_{out}$, $T_{pv}$, $\dot{m}$).
    *   `utils_matplotlib_plot_exergy.py`: Focuses strictly on the thermodynamic decomposition of thermal, electrical, and pumping exergy over time.
    *   `utils_matplotlib_plot_sensitivity.py`: Generates surface plots detailing system responsiveness to varying $T_a$ and $G$ spaces.
*   **OriginPro Publication Integration (`CORE_solar_fitting_origin.py` / `utils_origin_plot_transient.py`)**:
    *   Implements an automated Object Model (COM) bridge to **OriginPro 2025b**.
    *   Responsible for rendering high-fidelity, journal-ready graphs. It mitigates Matplotlib's layout limitations by leveraging Origin's advanced rendering engine for standard deviations, residual analysis, and multi-layer axis alignment.

## 5. File Structure & Conventions
*   `CORE_pvt_config.py`: Centralized physical and geometric parameters.
*   `CORE_pvt_simulation.py`: Primary simulation using steady-state analytical HWB logic.
*   `CORE_pvt_host_optimizer.py`: Global Exergy optimization and LUT generation boundary.
*   `CORE_pvt_mcu_control.c`: Reentrant MCU PID and Interpolation firmware (compiled to `pvt_mcu.dll`).
*   `CORE_pvt_independent_evaluator.py`: Cross-model validation (SIL) using an independent transient RC model.
*   `CORE_pvt_authenticity_monitor.py`: Stress test with synthetic cloud noise and sampling artifacts.

## 6. How to Run
1. Ensure `python` with `numpy`, `pandas`, `matplotlib`, and `gcc` (for DLL build) are installed.
2. Run `CORE_pvt_host_optimizer.py` to generate the controller LUT.
3. Run `gcc -shared -o pvt_mcu.dll CORE_pvt_mcu_control.c -lm` to compile the firmware.
4. Execute `CORE_pvt_independent_evaluator.py` for a comprehensive system audit.
