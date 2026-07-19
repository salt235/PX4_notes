# DIY无人机记录

## 组装与校准

### 组装

这一块我没有详细记录，主要是同组另一位同学记录组装的全过程，我这边就只贴几张过程图。

![uav](https://cdn.jsdelivr.net/gh/salt235/tuchuang/img/uav.png)

组装还是蛮容易的，这一块最难最重要的一部分是焊接，尝试了很久，后来终于有点熟练了。

![75e6b88e4c6daf6c72ee5ede96fe0532_0](https://cdn.jsdelivr.net/gh/salt235/tuchuang/img/75e6b88e4c6daf6c72ee5ede96fe0532_0.png)

我们同时组装了两台一样的无人机。

![961f1a98be6f21c0b62f217461f645d7_0](https://cdn.jsdelivr.net/gh/salt235/tuchuang/img/961f1a98be6f21c0b62f217461f645d7_0.png)

### 校准

这一部分我也没有过多记录，组装完成后，在QGC刷一下固件；传感器一栏里面把所有的传感器按照教程校准一遍；然后再把遥控器配对，把遥控器的遥感校准，频道也设置一下；接着再电源一栏里面把电调校准；之后再校准电机，设置四个电机，确认好转向；最后设置一下安全设置。

参考师兄的记录：https://app.notion.com/p/2f6b7664a095803da255c8785572a93d?v=2f6b7664a0958013a6a6000c337cbfa1&p=34fb7664a09580629c49edea0f275144&pm=s

## 试飞记录

### 第一次起飞（成功）

第一次简单飞了一下，感觉很好操控，很稳。但结果回来发现设置的是postion模式，本来就好操控。

### 第二次起飞（炸机）

第二次换成stable模式，一下子难操控很多，轻推油门直接一飞冲天，而且无人机一直在随风飘远，很难操控，最终降低油门无人机直线下降，撞坏了一个腿，分电板掉了一根焊线。

<img src="https://cdn.jsdelivr.net/gh/salt235/tuchuang/img/e77f3094ec336e1c44bf1672397f7674.png" alt="e77f3094ec336e1c44bf1672397f7674" style="zoom: 25%;" />

修好之后，重新把分电板的焊点加固了一下。

<img src="https://cdn.jsdelivr.net/gh/salt235/tuchuang/img/a33ac419707e10f21801561f14ab7de9.png" alt="a33ac419707e10f21801561f14ab7de9" style="zoom:25%;" />

### 第三次起飞（炸机）

想尝试一下大幅度的飞行，看看角加速度的曲线怎么样，便于后续调PID。这次操控也有经验一些了，大概能简单调整无人机的位置，但无人机的油门依旧很难操控，总是一飞冲天，还一直漂移。期间无人机飞的很高，得有十米了，而且会自动悬停在那个高度，这个时候降油门也没反应，只能赶快调到positon模式，再降低油门让无人机慢慢降下来。后来发现无人机当时好像自动切到一个loiter模式了，悬停在空中，可能是由于飞太高了吧。这次也有一点经验了，知道灵活切换到position模式来保活。

这次炸机应该是因为电机烧了，可能是飞太久了，在stable模式缓慢降落过程中，大概是又加了一点油门吧，突然电机狂转冒烟，无人机侧翻倒地，幸亏外观没有损坏，只是电源掉出来了，后来换了一个新的电调和电机。

<img src="https://cdn.jsdelivr.net/gh/salt235/tuchuang/img/4031743fa1449a36c29613ef02581249.png" alt="4031743fa1449a36c29613ef02581249" style="zoom: 67%;" />

### 第四次起飞（炸机）

油门依然很难操控，总是一窜上天。有时候还感觉推油门没反应，降油门也不降落，很奇怪，而且这几次无人机起飞都是三个电机先转，还有一个电机每次都是慢一拍开转。

这次炸机或许是因为无人机离我们水平位置太远了，也可能是接收器松动了，遥控器突然断了。无人机一直飞到中操的栏杆，撞上去了，然后翻在地上，电机转不动，遥控器也断了，kill不了。我们冒险去把无人机的电源拔了，无人机掉了一个桨叶，其他桨叶也磨了好多。同时遥控器接收器也断了，掉出来了，这次维修我们用3M胶和绝缘胶加固了接收器，也换了新桨叶。

<img src="https://cdn.jsdelivr.net/gh/salt235/tuchuang/img/507ec4b6cca4b3e3a089e8a54ab9f9cf.png" alt="507ec4b6cca4b3e3a089e8a54ab9f9cf" style="zoom:25%;" />

### 第五次起飞（成功）

在调PID之前，发现还需要先调一下滤波。于是这次先去飞个简单的日志，回来分析一下滤波图。这次飞行还是遇到之前的那个问题，就是无人机一飞冲天，然后降油门又不降落，最终还是强迫切换为position模式降落。

回来分析了一下滤波图，将低通滤波`IMU_DGYRO_CUTOFF`设为20，`IMU_GYRO_CUTOF`设为40；又将静态陷波滤波设为：

```
IMU_GYRO_NF0_FRQ = 80
IMU_GYRO_NF0_BW = 20
IMU_GYRO_NF1_FRQ =160
IMU_GYRO_NF1_BW =20
```

### 第六次起飞（成功）

调完滤波，又重新校准了一下PWM电调和加速度计。之前一解锁，要加油门电机才开始转，而且总有一个电机慢一拍，现在一解锁电机就直接开转了，师兄说这才是正常的。这次飞确实感觉稳了一些，没有一飞冲天，但是现在还是感觉油门还是有点灵敏，还是会不小心飞高。下次应该给遥控器油门安装一个回中配件，这样飞的稳一些。

这次主要还是看滤波的调整结果，结果还可以。

这是before：

![cfdb6bb1-8556-4cba-a956-2fcb09b1222c](https://cdn.jsdelivr.net/gh/salt235/tuchuang/img/cfdb6bb1-8556-4cba-a956-2fcb09b1222c.png)

这是after，明显看到陷波的两条线了，效果还是有的：

![f46a089609907c9576338cedfd203094](https://cdn.jsdelivr.net/gh/salt235/tuchuang/img/f46a089609907c9576338cedfd203094.png)

### 第七次起飞（成功）

![image-20260718134022633](https://cdn.jsdelivr.net/gh/salt235/tuchuang/img/image-20260718134022633.png)

![image-20260718134214614](https://cdn.jsdelivr.net/gh/salt235/tuchuang/img/image-20260718134214614.png)

分析了Roll和Pitch的角速度图像（上图）之后，尝试调参，将Roll的D从0.003加到0.0036，将Pitch的D从0.003加到0.0034。

![image-20260718134848315](https://cdn.jsdelivr.net/gh/salt235/tuchuang/img/image-20260718134848315.png)

![image-20260718134929943](https://cdn.jsdelivr.net/gh/salt235/tuchuang/img/image-20260718134929943.png)

这次起飞更有经验了，需要先接飞控，等遥控器连上了，再接电机。不然有可能遥控器还没有接上，电机就开转了，又控制不了。

最终结果是D仍然加得不够多，继续将Roll的D增加到0.0042，Pitch的D也增加到0.0042。

### 第八次试飞（成功）

![e7c816b23c745261159f27ed6445300a](https://cdn.jsdelivr.net/gh/salt235/tuchuang/img/e7c816b23c745261159f27ed6445300a.png)

Roll有一处异常，其余高度都不错。

![95741f588ad68669e7abfc5f28afc2b8](https://cdn.jsdelivr.net/gh/salt235/tuchuang/img/95741f588ad68669e7abfc5f28afc2b8.png)

Pitch有三处异常，看起来还是有一点超调。

但是分析下面两个阶跃图，感觉其实还行，Ptich的这个图相当好了，Roll的这个图也差不多了，但是可能还需要调整一下I。

![3ac1dde33cba70029626422fcc7ad027](https://cdn.jsdelivr.net/gh/salt235/tuchuang/img/3ac1dde33cba70029626422fcc7ad027.png)

![4f59b519ee3b3cb249076159007079bf](https://cdn.jsdelivr.net/gh/salt235/tuchuang/img/4f59b519ee3b3cb249076159007079bf.png)

同时感觉绿橙两条线距离太远了，这次准备先把K加20%，从1到1.2，再测试一次，先不改I，因为I会随着K的变化而增大。

### 第九次试飞（成功）

调完K之后又飞了一次，好像不应该先调K，现在这个阶跃图直接变到0.5了，于是我还把K调回去了，先调I吧。这次把Roll的i从0.2调到了0.24，加了20%。

![image-20260718161730118](https://cdn.jsdelivr.net/gh/salt235/tuchuang/img/image-20260718161730118.png)

这次遇到的问题是，阶跃图最下面有一段黑条，然后相应反而变差了，离1更远了。考虑到可能是飞行的问题，下次准备再空中先多悬停一会，然后再给各个方向的激励。最后降落也尽量稳一些，慢慢降落下来，不kill。

### 第十次试飞（大成功）

这次飞得非常稳，飞了很久，落地也稳，这个阶跃图正常了，所以说飞行的时候还是需要多飞多测一会。

![image-20260718164113630](https://cdn.jsdelivr.net/gh/salt235/tuchuang/img/image-20260718164113630.png)

![image-20260718164142387](https://cdn.jsdelivr.net/gh/salt235/tuchuang/img/image-20260718164142387.png)

可以看到，Pitch的图依旧很稳，然后Roll的图这次超过了一点，于是我们把Roll的I从0.24降低到0.23了。

关于Yaw，我们这个貌似非常好，不用再调了。

![image-20260718165550008](https://cdn.jsdelivr.net/gh/salt235/tuchuang/img/image-20260718165550008.png)

### 第十一次试飞（）

![image-20260718170700391](https://cdn.jsdelivr.net/gh/salt235/tuchuang/img/image-20260718170700391.png)

![image-20260718170727592](https://cdn.jsdelivr.net/gh/salt235/tuchuang/img/image-20260718170727592.png)

现在开始调角度环了，发现Roll的图像还可以，Pitch需要稍微加一点P。

于是我们把Pitch的P从6.5加到了7，看看效果。
