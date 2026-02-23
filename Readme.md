<div align="center">
  <h1>Go2 Intelligence Framework</h1>
  <p>ROS 2 & Isaac Sim based intelligence framework for Unitree Go2.</p>
</div>

## 🗺️ Project Roadmap
This project aims to build a comprehensive intelligence framework for the Unitree Go2 robot. We are planning to expand the framework step-by-step.

- [x] **3D SLAM (RTAB-Map)**: Visual and Depth based mapping in Isaac Sim.
- [ ] **Navigation (Nav2)**: Autonomous path planning and obstacle avoidance.
- [ ] **Reinforcement Learning**: Advanced locomotion and task-specific policy training.
- [ ] **Real-world Deployment**: Sim2Real transfer and deployment on physical Go2 hardware.

---

## 🏗️ Modules

### 1. 3D SLAM (RTAB-Map in Isaac Sim)
Demonstrates 3D environmental mapping using RTAB-Map with the Go2 robot within the NVIDIA Isaac Sim environment.

#### 🎥 Demonstration Video
<div align="center">
  <a href="https://youtu.be/ZbYe7EWJfB8">
    <img src="https://img.youtube.com/vi/ZbYe7EWJfB8/0.jpg" alt="RTAB-Map SLAM Demonstration" width="600">
  </a>
  <p><i>Click the image to watch the RTAB-Map SLAM demonstration in action.</i></p>
</div>

#### 💻 Quick Start
To run the full simulation and SLAM pipeline, please open three separate terminals.

**Terminal A**: Start the Go2 simulation environment
```bash
cd ~/Desktop/sj/go2_intelligence_framework
python scripts/go2_sim.py
```

**Terminal B**: Launch the RTAB-Map SLAM node
```bash
cd ~/Desktop/sj/go2_intelligence_framework
ros2 launch launch/go2_rtabmap.launch.py
```

**Terminal C**: Open RViz Visualization
```bash
cd ~/Desktop/sj/go2_intelligence_framework
rviz2 -d config/go2_sim.rviz
```

---

### 2. Autonomous Navigation (Nav2)
*Coming Soon: Integration with ROS 2 Nav2 stack for autonomous waypoint navigation.*

---

### 3. Reinforcement Learning
*Coming Soon: RL environment setup and policy training for Go2 locomotion and intelligent behavior.*

---

### 4. Real-world Deployment
*Coming Soon: Guidelines and scripts for deploying the developed intelligence on the actual Unitree Go2 robot.*
