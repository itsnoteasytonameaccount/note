---
title: 互斥锁
tags:
- 算法
- c
date: 2020/05/02
---
# 前言
互斥锁方面算法，因为毕设有要用到的地方，所以复习了一下，记录成果
<!-- more -->

## 例1
### 程序
```c++
int lock = 0;
void progress(){
    while(lock) ;
    lock = 1; // (1)
    //dosomething
    lock = 0;
}
```
### 问题
1.  可能多个进程都过临界

## 例2
### 程序
```c++
int turn = 0;
void progress0(){
    //turn = 1; //多此一举
    while(turn == 1);
    turn = 0; //(1)
    //dosomething
    turn = 1;
}

void progesss1(){
    //turn == 0; //多此一举
    while(turn == 0);
    turn = 1; //(1)
    //dosomething
    turn = 0; //(2)
}
```
### 问题
1.  同时只能有1个过临界，但进临界后必须等另一个进一次临界才能再次进临界，只有一个出了临界，另一个才能进临界，必须交替执行
2.  步骤(1)可以无，此处方便理解，步骤(1)上移可能出现两者都过临界
3.  若turn可以有3个值，turn == 2代表都未到达临界，此时把步骤(2)改为turn = 2，则可能出现两者都过临界，同例1


## 例3
### 程序
```c++
int flag[2] = {0, 0};
void progress0(){
    while(flag[1]); //(1)
    flag[0] = 1; //(2)
    //dosomething
    flag[0] = 0;
}

void progress1(){
    while(flag[0]); //(1)
    flag[1] = 1; //(2)
    //dosomething
    flag[1] = 0;
}
```
### 问题
1.  同例1
2.  把(2)移至(1)前可能会死锁

## dekker算法
### 程序
```c++
int flag[2] = {0, 0};
int turn = 0;
void progress0(){
    flag[0] = 1;        //(1) 执行完此处，另一个不可以过临界
    while(flag[1]){     //(2) 分析1
        if(turn == 1){  //(3) 分析2
            flag[0] = 0;//(4）
            while(turn == 1);
            flag[0] = 1;//(5)
        }
    }
    //dosomething
    turn = 1;
    flag[0] = 0;
}

void progress1(){
    flag[1] = 1;        //(1)
    while(flag[0]){     //(2)
        if(turn == 0){  //(3)
            flag[1] = 0;//(4)
            while(turn == 0);
            flag[1] = 1;//(5)
        }
    }
    //dosomething
    turn = 0;
    flag[1] = 0;
}
```
### 分析
1.  三种
    >当进程1执行完步骤(1),执行外层循环内部代码
    >当进程1未执行完步骤(1),直接访问
    >当进程1进入内层循环时，跳出循环
2.  两种
    >当两者都执行至此，根据turn值不同一个会跳出循环，一个进入内层循环
    >当进程1已开始访问，而turn又不等于1时，进程0会一直执行外部循环判断
3.  唯一出现多等待一会的情况：进程0执行一遍后，进程1执行完内层循环，还未执行完步骤(5)，进程0又执行完步骤(2)

## Peterson`s 算法
```c++
int flag[2] = {0,0};
int turn = 0;

void progress0(){
    flag[0] = 1; //(1)
    turn = 1; //(2)
    while(flag[1] && turn == 1); //(3)
    //dosomething
    flag[0] = 0; //(4)
}

void progress1(){
    flag[1] = 1; //(1)
    turn = 0; //(2)
    while(flag[0] && turn == 0); //(3)
    //dosomething
    flag[1] = 0; //(4)
}
```
### 分析
1.  当进程0执行到步骤(3),而进程1未执行完(1),直接访问
2.  当进程0执行到步骤(3),进程1未执行完步骤(2),在进程1执行完步骤(2)之后,进程0优先访问
3.  当进程0执行到步骤(3),进程1也执行到步骤(3),访问权取决于turn最后的值
4.  当进程0执行到步骤(3),进程1已开始访问，此时flag[1] = 1,turn = 1必然成立
5.  当进程0执行到步骤(3),进程1执行完1遍,回到分析1
