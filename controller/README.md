<!--
 * @Author: jinde.zgm
 * @Date: 2021-03-26 21:49:22
 * @Description: Kubernetes的controller源码解析
-->

# 名词解释

1. XxxController: 泛指各种Workload控制器，比如DeploymentController、DeamonSetController、StatefulSetController等等；
2. Xxx: 泛指各种Workload，比如Deployment、DeamonSet、StatefulSet等等；
3. Owner: Kubernetes的API对象的一个meta属相，即对象的拥有者，也称之为父对象，该API对象也称为Owner的子对象；
4. Controller: XxxController的功能是让系统达到Xxx声明的状态，XxxController负责执行创建、删除子对象操作；换个角度看，Xxx才是子对象的控制器，无非由XxxController代为执行而已(用C/C++视角，XxxController就是Xxx的线程池+静态成员)，所以在很多代码中Controller指的就是Xxx，而不是XxxController，泛指Xxx是其子对象的控制器；

# 目录

1. [PodControl](./PodControl.md)
2. [ControllerExpectations](./ControllerExpectations.md)
