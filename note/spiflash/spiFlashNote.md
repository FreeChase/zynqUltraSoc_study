

# SPIFLASH调试笔记

1. The effect of the command is immediate. The default address mode is three bytes, and the device returns to the default upon exiting the 4-byte address mode

## 命令


|命令|代码|备注|
|-|-|-|
| ENTER 4-BYTE ADDRESS MODE|B7h||
| EXIT 4-BYTE ADDRESS MODE|E9h||
