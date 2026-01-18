# kgpinn-with-ddpg
This project presents a control-oriented thermal modeling and demand response framework for modern data centers, where HVAC systems must operate under highly dynamic and spatially heterogeneous IT workloads. The proposed approach integrates Knowledge Graph (KG) modeling, Physics-Informed Neural Networks (PINNs), and Deep Reinforcement Learning (DDPG) to jointly address prediction accuracy, physical interpretability, and real-time control robustness.

The framework employs a KG to encode spatial structure, airflow topology, and empirical thermodynamic relationships, which are then embedded into a PINN surrogate to model high-dimensional thermo-fluid dynamics with improved convergence and extrapolation capability. Building on this surrogate, a DDPG-based controller enables adaptive multi-input–multi-output (MIMO) HVAC control, achieving stable temperature regulation and energy efficiency under rapid workload fluctuations. Experimental validation on two GPU halls and one CPU hall in a large-scale production data center demonstrates consistent improvements over conventional PID control in both thermal stability and control robustness.

The current implementation and validation focus on an air-cooled, GPU-centric facility in a cold and dry climate. While extension to other climates and cooling architectures will require site-specific recalibration, the modular KG-PINN + RL design provides a scalable foundation for future developments, including multi-agent RL, meta-learning, and integration with digital-twin or BIM platforms.

Data and code availability.
This project is conducted under an Alibaba Cloud academic collaboration agreement. Due to data confidentiality constraints, the public GitHub repository provides:

Python runtime code for model inference and control,

Pre-trained and packaged FMU files generated after training.

Raw training and validation datasets are not publicly released but may be made available under limited disclosure upon reasonable academic request via email.
