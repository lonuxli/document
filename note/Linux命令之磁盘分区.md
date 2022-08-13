# Linux命令之磁盘分区

parted命令使用

|print  \[free|all | NUMBER\]               | 查看分区状态信息                                                                                                                                      |
|-------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------|
|mkpart PART\-TYPE START END                |PART\-TYPE: primary  extended   logical 

START, END  开始，结束为止
 创建新分区，                                                                        |
|set NUMBER  FLAG  STATE                    |FLAG: boot  引导， hidden 隐藏， raid  软raid， lvm  逻辑卷， 

STATE:  on| off
 对编号为NUMBER的进行标记。                                               |
|mkfs NUMBER FS\-TYPE                       | 对NUMBER指定文件系统。FS\-Type有：ext2、fat16、fat32、linuxswap、NTFS、reiserfs、ufs 等                                                               |
| cp  \[FROM\-DEV\] FROM\-NUMBER  TO\-NUMBER| 将分区 FROM\-NUMBER 上的文件系统完整地复制到分区TO\-NUMBER 中，作为可选项还可以指定一个来源硬盘的设备名称FROM\-DEVICE，若省略则在当前设备上进行复制。 |
| move NUMBER START END                     | 将指定编号 NUMBER 的分区移动到从 START 开始 END 结束的位置上。注意：（1）只能将分区移动到空闲空间中。（2）虽然分区被移动了，但它的分区编号是不会改变的|
|resize NUMBER START END　　                |对指定编号 NUMBER 的分区调整大小。分区的开始位置和结束位置由 START 和 END 决定                                                                         |
|check NUMBER                               |检查指定编号 NUMBER 分区中的文件系统是否有什么错误                                                                                                     |
|rescue START END                           |rescue START END                                                                                                                                       |
|mklabel,mktable LABELTYPE                  |创建一个新的 LABEL\-TYPE 类型的空磁盘分区表，对于PC而言 msdos 是常用的 LABELTYPE。 若是用 GUID 分区表，LABEL\-TYPE 应该为 gpt                          |

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 
