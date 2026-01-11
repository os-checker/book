# UPG - Web UI

## 概念

Unsafety Propagation Graph (UPG) 详细概念介绍见 RAPx book
《[Unsafe Code Audit](https://artisan-lab.github.io/RAPx-Book/6.4-unsafe.html)》。

要点：

* Endogenous：指直接的不安全操作（比如调用不安全函数、解引用裸指针、访问 `static mut` 
  等）；作为安全属性的内生性，它天然地具有 checked、delegated 等操作。
* Exogenous：指影响被调用的不安全函数的情况（比如构造函数、具有可变性的方法等）；
  作为安全属性的外生性，该影响没有直接反映在调用上。比如参数的类型（或结构体字段）
  不变量潜在地被其他方式修改，从而安全属性的满足需要考虑这些情况。


## 设计草图

<div style="text-align: center;">
<a href="https://excalidraw.com/#json=xYOQhebscx19YcW5e0n2Y,r_0gIGIhJM09AHxo492GFw" target="_blank">
<img src="https://github.com/user-attachments/assets/a79fc968-c78c-465c-93a4-0610696af35a" />
</a>

<a href="https://excalidraw.com/#json=MGE2yKBk_N65o4lw3rwWW,8rW9f7u6-Fl0KEqdX2Silg" target="_blank">
<img src="https://github.com/user-attachments/assets/58ffbe00-edeb-4f83-b447-70a53406ea93" />
</a>
</div>

