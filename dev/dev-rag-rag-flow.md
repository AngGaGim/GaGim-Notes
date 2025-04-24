[[2025-04-23 14:32]]
## bug

### 1. ocr识别错误

- 原文中的句号识别为‘G’

![[Pasted image 20250423143353.png]]
  
![[Pasted image 20250423143316.png]]

原文为练字

![[Pasted image 20250424143932.png]]


原文为工作
![[Pasted image 20250424144109.png]]


### 2. token分块，一句话被两个分块截断
#### 解决方法：
遍历每一个section，再遍历每个section的每个字符串，如果遇到标点符号，则当前分块截断并结束。


### 3.标题未区分

![[Pasted image 20250424144422.png]]
![[Pasted image 20250424144436.png]]



### 4. 文本缺失

![[Pasted image 20250424144726.png]]

![[Pasted image 20250424144742.png]]