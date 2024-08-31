# 远程攻击的怪物（RangedAttackMob）

下面是远程攻击的怪物，这一部分我们分两块研究。制作一个实现`RangedAttackMob`接口的远程攻击的怪物相对简单，只需要为你的怪物添加一个AI，并实现该接口内的抽象方法`performRangedAttack`就可以。

添加的AI一般是`RangedAttackGoal`，但如果你的怪物使用弓，则应添加`RangedBowAttackGoal`，如果怪物使用弩，则不仅AI要改变，实现的接口也要变为继承了`RangedAttackMob`的`CrossbowAttackMob`。

接下来我们将围绕`performRangedAttack`方法出发，先熟悉这种最基础的远程攻击方式的实现。
