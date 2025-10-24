# 需求文档 - PSD批量换字工具

## 简介

本系统是一个智能化的Web应用，旨在自动化处理PSD模板文件的批量文字替换需求。用户上传PSD模板文件和文字列表后，系统自动识别目标图层、提取字体参数、生成预览，并批量生成最终图片。整个过程通过智能匹配、OCR识别、交互式预览和实时反馈，为用户提供高效、可控的批量图片生成体验。

## 术语表

- **System**: PSD批量换字工具系统
- **User**: 使用本系统的设计师或运营人员
- **PSD文件**: Adobe Photoshop设计文件
- **图层 (Layer)**: PSD文件中的独立可编辑元素
- **Type图层**: PSD中包含可编辑文字的图层类型
- **Pixel图层**: PSD中已栅格化的像素图层
- **字体包 (Font Package)**: 用户上传的字体文件，支持TTF、OTF、TTC格式
- **字体标识符 (Font ID)**: 系统为上传的字体文件生成的唯一标识
- **文字列表 (Text List)**: 用户提供的需要批量替换的文本内容列表，可通过手动输入、Excel或CSV文件提供
- **任务ID (Task ID)**: 系统为每个批量生成任务分配的唯一标识符
- **目标图层 (Target Layer)**: 被选中用于文字替换的图层
- **原始文本 (Original Text)**: 目标图层中的初始文字内容
- **边界框 (Bbox)**: 图层的位置和尺寸信息 (x, y, width, height)
- **背景板 (Background Plate)**: 除目标图层外所有可见图层的合成图像
- **Demo预览**: 使用第一个替换文本生成的示例图片
- **WebSocket**: 用于服务器向客户端实时推送数据的通信协议
- **缩略图 (Thumbnail)**: 低分辨率的预览图片
- **成品图**: 全分辨率的最终输出图片
- **云存储 (Cloud Storage)**: 用于存储成品图的远程存储服务（如R2）
- **预签名URL (Pre-signed URL)**: 带有临时访问权限的云存储下载链接
- **OCR**: 光学字符识别技术，用于从图像中提取文字
- **二分法 (Binary Search)**: 用于反推字体大小的算法
- **字号锁定 (Font Size Locking)**: 通过算法确定的最佳字体大小

## 需求

### 需求 1: PSD文件上传和解析

**用户故事**: 作为设计师，我希望能够上传PSD模板文件，以便系统能够解析其中的图层信息。

#### 验收标准

1. WHEN User上传一个PSD文件，THE System SHALL接受该文件并开始解析
2. WHEN System解析PSD文件时，THE System SHALL提取所有图层的元数据，包括图层名称、类型、边界框和可见性
3. WHEN System完成PSD解析时，THE System SHALL缓存所有图层的元数据供后续使用
4. IF PSD文件格式无效或损坏，THEN THE System SHALL返回明确的错误信息给User
5. THE System SHALL支持包含中文图层名称的PSD文件

### 需求 2: 字体包上传

**用户故事**: 作为设计师，我希望能够上传自定义字体文件，以便系统在生成图片时使用正确的字体渲染文字。

#### 验收标准

1. THE System SHALL支持上传TTF、OTF和TTC格式的字体文件
2. WHEN User上传字体文件时，THE System SHALL验证文件格式的有效性
3. WHEN System接收字体文件时，THE System SHALL将字体文件临时存储在服务器
4. THE System SHALL为每个上传的字体文件生成唯一的标识符
5. WHEN System完成字体上传时，THE System SHALL返回字体标识符供后续使用
6. THE System SHALL在任务完成后自动清理临时存储的字体文件

### 需求 3: 文字列表输入

**用户故事**: 作为设计师，我希望能够通过多种方式提供文字列表（手动输入、上传Excel或CSV文件），以便灵活地准备批量生成所需的数据。

#### 验收标准

1. THE System SHALL支持三种文字列表输入方式：手动文本输入、Excel文件上传、CSV文件上传
2. WHEN User选择手动输入时，THE System SHALL提供文本输入区域供User粘贴字符串列表
3. WHEN User手动输入字符串列表时，THE System SHALL解析每一行作为一个独立的替换文本
4. WHEN User上传Excel或CSV文件时，THE System SHALL读取文件内容并提取文字列表
5. WHEN System读取表格文件时，THE System SHALL按照指定列或默认第一列提取文字数据
6. WHEN System处理表格文件时，THE System SHALL对提取的文字列表进行排序
7. WHEN System完成文字列表处理时，THE System SHALL将列表数据存储到数据库并生成任务ID
8. THE System SHALL支持包含特殊字符和多语言文字的字符串
9. WHEN User提交文字列表时，THE System SHALL验证列表不为空
10. THE System SHALL显示文字列表中的条目数量和任务ID

### 需求 4: 智能图层自动匹配

**用户故事**: 作为设计师，我希望系统能够自动找到需要替换文字的图层，以便节省手动选择的时间。

#### 验收标准

1. WHEN System接收到字符串列表时，THE System SHALL自动搜索所有Type图层
2. WHEN System搜索Type图层时，THE System SHALL识别图层文本内容是否存在于字符串列表中
3. IF System找到唯一匹配的Type图层，THEN THE System SHALL自动锁定该图层作为目标图层
4. WHEN System锁定目标图层时，THE System SHALL提取该图层的原始文本和边界框信息
5. IF System找到唯一匹配的图层，THEN THE System SHALL跳过手动选择步骤，直接进入参数提取流程

### 需求 5: 手动图层选择（容错机制）

**用户故事**: 作为设计师，当系统无法自动识别目标图层时，我希望能够手动选择，以便确保流程能够继续。

#### 验收标准

1. IF System未找到匹配的图层或找到多个匹配图层，THEN THE System SHALL向User展示所有可见图层的列表
2. WHEN System展示图层列表时，THE System SHALL显示每个图层的名称和类型（Type或Pixel）
3. THE System SHALL允许User从列表中选择一个图层作为目标图层
4. WHEN User选择一个图层时，THE System SHALL记录该选择并继续处理流程
5. THE System SHALL提供图层预览缩略图帮助User识别正确的图层

### 需求 6: OCR文字识别（栅格化图层处理）

**用户故事**: 作为设计师，当我选择的图层是栅格化的像素图时，我希望系统能够识别其中的文字，以便继续使用该图层。

#### 验收标准

1. WHEN User选择一个Pixel图层时，THE System SHALL导出该图层为PNG图像
2. WHEN System导出Pixel图层时，THE System SHALL使用OCR技术识别图像中的文字内容
3. WHEN System完成OCR识别时，THE System SHALL向User展示识别结果和图层预览图
4. THE System SHALL允许User确认或修正OCR识别的文本内容
5. WHEN User确认文本内容时，THE System SHALL将该文本作为原始文本继续处理

### 需求 7: 字体参数提取

**用户故事**: 作为设计师，我希望系统能够自动计算出正确的字体大小和颜色，以便生成的图片与原始设计保持一致。

#### 验收标准

1. WHEN System获得原始文本和边界框信息时，THE System SHALL使用二分查找算法反推最佳字体大小
2. THE System SHALL确保反推的字体大小能够使文字完整显示在边界框内
3. WHEN System处理Type图层时，THE System SHALL尝试从图层元数据中提取原始文字颜色
4. IF System无法提取文字颜色，THEN THE System SHALL使用黑色作为默认颜色
5. WHEN System完成参数提取时，THE System SHALL生成背景板图像（排除目标图层的所有可见图层合成）

### 需求 8: 交互式Demo预览

**用户故事**: 作为设计师，我希望在批量生成前能够看到一个示例效果，并能够调整文字颜色，以便确认最终效果符合预期。

#### 验收标准

1. WHEN System完成参数提取时，THE System SHALL使用字符串列表的第一个文本生成Demo预览图
2. THE System SHALL在Demo预览图中使用锁定的字体大小和提取的默认颜色
3. THE System SHALL向User展示Demo预览图和颜色选择器
4. WHEN User更改颜色选择器时，THE System SHALL实时重新生成Demo预览图
5. THE System SHALL在Demo预览中使用缓存的背景板以提高响应速度

### 需求 9: 批量生成参数确认

**用户故事**: 作为设计师，我希望能够确认最终的生成参数，包括颜色和裁剪选项，以便开始批量生成。

#### 验收标准

1. THE System SHALL提供"开始批量生成"按钮供User确认参数
2. THE System SHALL提供可选的"裁剪边缘空白"选项
3. WHEN User点击"开始批量生成"时，THE System SHALL记录User确认的最终颜色
4. WHEN User点击"开始批量生成"时，THE System SHALL记录User选择的裁剪选项
5. WHEN User确认参数时，THE System SHALL建立WebSocket连接准备实时推送

### 需求 10: 批量图片生成

**用户故事**: 作为设计师，我希望系统能够自动生成所有版本的图片，以便我不需要手动重复操作。

#### 验收标准

1. WHEN System开始批量生成时，THE System SHALL遍历字符串列表中的每个文本
2. WHEN System处理每个文本时，THE System SHALL使用锁定的字体大小和确认的颜色绘制文字
3. WHEN System绘制文字时，THE System SHALL将文字图层与背景板合成生成最终图像
4. IF User选择了裁剪选项，THEN THE System SHALL裁剪图像的边缘空白区域
5. WHEN System生成每张图片时，THE System SHALL将全分辨率成品图上传到云存储

### 需求 11: 实时进度反馈

**用户故事**: 作为设计师，我希望能够实时看到生成进度，以便了解任务完成情况而不是焦虑等待。

#### 验收标准

1. WHEN System生成每张成品图时，THE System SHALL创建该图片的缩略图
2. WHEN System创建缩略图时，THE System SHALL通过WebSocket推送给User
3. THE System SHALL在User界面上实时显示接收到的缩略图
4. THE System SHALL按照生成顺序展示缩略图
5. WHEN System完成所有图片生成时，THE System SHALL关闭WebSocket连接并通知User

### 需求 12: 批量下载

**用户故事**: 作为设计师，我希望能够一键下载所有生成的图片，以便快速获取成果。

#### 验收标准

1. WHEN System完成所有图片生成时，THE System SHALL显示"下载"按钮
2. WHEN User点击"下载"按钮时，THE System SHALL生成云存储的预签名Zip下载链接
3. THE System SHALL确保预签名URL包含本次任务的所有成品图
4. WHEN System生成下载链接时，THE System SHALL触发User浏览器开始下载
5. THE System SHALL确保下载流量直接来自云存储而非应用服务器

### 需求 13: 错误处理和用户反馈

**用户故事**: 作为设计师，当系统遇到问题时，我希望能够收到清晰的错误提示，以便知道如何解决问题。

#### 验收标准

1. IF System在任何步骤遇到错误，THEN THE System SHALL向User显示明确的错误信息
2. WHEN System显示错误信息时，THE System SHALL说明错误原因和可能的解决方案
3. IF 批量生成过程中某张图片失败，THEN THE System SHALL继续处理其余图片并记录失败项
4. THE System SHALL在批量生成完成后显示成功和失败的统计信息
5. THE System SHALL提供重试失败项的选项

### 需求 14: 性能和可扩展性

**用户故事**: 作为设计师，我希望系统能够快速处理大量图片，以便提高工作效率。

#### 验收标准

1. THE System SHALL在5秒内完成PSD文件解析和图层元数据提取
2. THE System SHALL在2秒内完成Demo预览图的生成和颜色更新
3. WHEN System进行批量生成时，THE System SHALL使用并行处理提高生成速度
4. THE System SHALL支持至少100张图片的批量生成任务
5. THE System SHALL在批量生成过程中保持WebSocket连接稳定

### 需求 15: 数据安全和隐私

**用户故事**: 作为设计师，我希望我的设计文件和生成的图片是安全的，以便保护商业机密。

#### 验收标准

1. THE System SHALL为每个User会话生成唯一的任务标识符
2. THE System SHALL确保不同User的任务数据相互隔离
3. THE System SHALL在任务完成后的24小时内自动清理临时文件
4. THE System SHALL使用带有时效性的预签名URL限制下载访问
5. THE System SHALL在云存储中使用任务标识符组织文件，防止路径猜测攻击
