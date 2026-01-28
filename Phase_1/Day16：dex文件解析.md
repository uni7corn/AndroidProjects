这一天实际上会进行很久，可能一周还可能更久
我选择看一下安卓源码里面的`dexdump`原生工具，从那个里面看 dex 文件，当然也肯定会看真 DexFile 的定义的

---

## dexdump.h

首先来看一个最基本也是最简单的，`dexdump.h`文件

```c
// 保护宏，这个我也不知道怎么形容了，可以问问AI为什么要这样写
#ifndef ART_DEXDUMP_DEXDUMP_H_
#define ART_DEXDUMP_DEXDUMP_H_

// 正常引入需要的文件
#include <stdint.h>
#include <stdio.h>

// 归属于art，所以写在art的命名空间内
namespace art
{

/* Supported output formats. */
// 输出文件的格式，输出到基本命令行还是输出为xml文件，默认是命令行文本
enum OutputFormat
{
	OUTPUT_PLAIN = 0, // default
	OUTPUT_XML, // XML-style
};

/* Command-line options. */
// 命令行配置结构，含义我写到下面
struct Options
{
	bool checksumOnly;
	bool disassemble;
	bool exportsOnly;
	bool ignoreBadChecksum;
	bool disableVerifier;
	bool showAnnotations;
	bool showCfg;
	bool showFileHeaders;
	bool showSectionHeaders;
	bool verbose;
	OutputFormat outputFormat;
	const char *outputFileName;
};

/* Prototypes. */
// 全局的变量，至于为什么这样写我觉得可以问问AI，也可以问问别的✋子
extern struct Options gOptions;
extern FILE *gOutFile;
// 处理单个文件的接口
int processFile(const char *fileName);

} // namespace art

#endif // ART_DEXDUMP_DEXDUMP_H_
// 宏保护结尾
```

工具的配置参数

| 成员                 | 含义                                             |
| -------------------- | ------------------------------------------------ |
| `checksumOnly`       | 只验证 dex 文件校验和                            |
| `disassemble`        | 反汇编 dex 中的 bytecode（查看 dalvik/ART 指令） |
| `exportsOnly`        | 仅输出导出信息（如类、方法、字段）               |
| `ignoreBadChecksum`  | 忽略校验和错误的 dex 文件                        |
| `disableVerifier`    | 不进行字节码验证                                 |
| `showAnnotations`    | 输出注解信息                                     |
| `showCfg`            | 输出控制流图（CFG）                              |
| `showFileHeaders`    | 输出 dex 文件头信息                              |
| `showSectionHeaders` | 输出各个 section 的头信息                        |
| `verbose`            | 输出详细日志                                     |
| `outputFormat`       | 输出格式，参考上面的枚举                         |
| `outputFileName`     | 指定输出文件，不指定则输出到标准输出             |

---

## dexdump.cc

我们接着看下一个`dexdump.cc`文件，这个文件是命令行部分的工作

```cpp
int main(int argc, char **argv)
{
	// Output all logging to stderr.
	android::base::SetLogger(android::base::StderrLogger);

	return art::dexdumpDriver(argc, argv);
}
```

我们来从主函数看，不过第一句也不用看，设置日志的，看第二个函数`dexdumpDriver`

```cpp
int dexdumpDriver(int argc, char **argv)
{
    // 1. 初始化
	memset(&gOptions, 0, sizeof(gOptions));
	gOptions.verbose = true;

    // 2. 解析命令行参数
	while (true) {
		const int ic = getopt(argc, argv, "acdefghijl:o:");
		if (ic < 0) break;
		switch (ic) {
		case 'a': gOptions.showAnnotations = true; break;
		case 'c': gOptions.checksumOnly = true; break;
		case 'd': gOptions.disassemble = true; break;
		case 'e': gOptions.exportsOnly = true; break;
		case 'f': gOptions.showFileHeaders = true; break;
		case 'g': gOptions.showCfg = true; break;
		case 'h': gOptions.showSectionHeaders = true; break;
		case 'i': gOptions.ignoreBadChecksum = true; break;
		case 'j': gOptions.disableVerifier = true; break;
		case 'l': // 输出格式
			if (strcmp(optarg, "plain") == 0) gOptions.outputFormat = OUTPUT_PLAIN;
			else if (strcmp(optarg, "xml") == 0) {
				gOptions.outputFormat = OUTPUT_XML;
				gOptions.verbose = false; // XML 输出不需要日志
			} else wantUsage = true;
			break;
		case 'o': gOptions.outputFileName = optarg; break;
		default: wantUsage = true; break;
		}
	}
	if (optind == argc) {
		LOG(ERROR) << "No file specified";
		wantUsage = true;
	}
	if (gOptions.checksumOnly && gOptions.ignoreBadChecksum) {
		LOG(ERROR) << "Can't specify both -c and -i";
		wantUsage = true;
	}
	if (wantUsage) {
		usage();
		return 2;
	}
	if (gOptions.outputFileName) {
		gOutFile = fopen(gOptions.outputFileName, "w");
		if (!gOutFile) {
			PLOG(ERROR) << "Can't open " << gOptions.outputFileName;
			return 1;
		}
	}
	int result = 0;
	while (optind < argc) {
		result |= processFile(argv[optind++]);
	}
	return result != 0 ? 1 : 0;
}
```

我自己手动给删除了点注释，如果看的不喜欢可以自己格式化一下

- getopt 标准命令行解析
- `wantUsage = true` 表示参数错误，之后会打印 usage 并退出
  下面的判断语句是解析命令行的，确保有一个 dex 文件加载解析，校验 i 和 c 这两个会不会同时出现，避免程序错误
  还有就是参数错误去打印错误并退出
  最后循环是便利 dex 文件，因为前面的我们都解析了，剩下的就只能是 dex 文件列表了，然后去给`processFile`

### processFile

这个函数就是解析了

```cpp
int processFile(const char *fileName)
{
	if (gOptions.verbose) {
		fprintf(gOutFile, "Processing '%s'...\n", fileName);
	}

	const bool kVerifyChecksum = !gOptions.ignoreBadChecksum;
	const bool kVerify = !gOptions.disableVerifier;
	std::string content;
	if (!android::base::ReadFileToString(fileName, &content)) {
		LOG(ERROR) << "ReadFileToString failed";
		return -1;
	}
	const DexFileLoader dex_file_loader;
	DexFileLoaderErrorCode error_code;
	std::string error_msg;
	std::vector<std::unique_ptr<const DexFile> > dex_files;
	if (!dex_file_loader.OpenAll(reinterpret_cast<const uint8_t *>(content.data()),
		    content.size(),
		    fileName,
		    kVerify,
		    kVerifyChecksum,
		    &error_code,
		    &error_msg,
		    &dex_files)) {
		LOG(ERROR) << error_msg;
		return -1;
	}
	if (gOptions.checksumOnly) {
		fprintf(gOutFile, "Checksum verified\n");
	} else {
		// Open XML context.
		if (gOptions.outputFormat == OUTPUT_XML) {
			fprintf(gOutFile, "<api>\n");
		}

		for (size_t i = 0, n = dex_files.size(); i < n; i++) {
			processDexFile(fileName, dex_files[i].get(), i, n);
		}

		// Close XML context.
		if (gOptions.outputFormat == OUTPUT_XML) {
			fprintf(gOutFile, "</api>\n");
		}
	}
	return 0;
}
```

`ReadFileToString`是读取文件，把文件名的内容读到上下文，接下里就是加载 dex 文件了`DexFileLoader`，是 dex 文件解析器
后面那个全部打开的方法是尝试解析全部 dex 文件，成功和失败就会按情况返回，结果是写入到上面的`dex_files`文件里面
接下来就是一个判断，如果命令行参数只需要计算校验和拿就输出然后返回，如果没说就接着去解析，不管是输出命令行还是 xml
文件，都会走到`processDexFile`函数

### processDexFile

我们来看这个函数内容

```cpp
static void processDexFile(const char *fileName, const DexFile *pDexFile, size_t i, size_t n)
{
	if (gOptions.verbose) {
		fputs("Opened '", gOutFile);
		fputs(fileName, gOutFile);
		if (n > 1) {
			fprintf(gOutFile, ":%s", DexFileLoader::GetMultiDexClassesDexName(i).c_str());
		}
		fprintf(gOutFile, "', DEX version '%.3s'\n", pDexFile->GetHeader().magic_ + 4);
	}

	// Headers.
	if (gOptions.showFileHeaders) {
		dumpFileHeader(pDexFile);
	}

	// Iterate over all classes.
	char *package = nullptr;
	const u4 classDefsSize = pDexFile->GetHeader().class_defs_size_;
	for (u4 j = 0; j < classDefsSize; j++) {
		dumpClass(pDexFile, j, &package);
	} // for

	// Iterate over all method handles.
	for (u4 j = 0; j < pDexFile->NumMethodHandles(); ++j) {
		dumpMethodHandle(pDexFile, j);
	} // for

	// Iterate over all call site ids.
	for (u4 j = 0; j < pDexFile->NumCallSiteIds(); ++j) {
		dumpCallSite(pDexFile, j);
	} // for

	// Free the last package allocated.
	if (package != nullptr) {
		fprintf(gOutFile, "</package>\n");
		free(package);
	}
}
```

这个看起来不多，实际上每个函数多麻了
最开头的判断还是输出文件名，文件版本之类的，无需在意

#### 头文件头信息

```cpp
static void dumpFileHeader(const DexFile *pDexFile)
{
	const DexFile::Header &pHeader = pDexFile->GetHeader();
	char sanitized[sizeof(pHeader.magic_) * 2 + 1];
	fprintf(gOutFile, "DEX file header:\n");
	asciify(sanitized, pHeader.magic_, sizeof(pHeader.magic_));
	fprintf(gOutFile, "magic               : '%s'\n", sanitized);
	fprintf(gOutFile, "checksum            : %08x\n", pHeader.checksum_);
	fprintf(gOutFile, "signature           : %02x%02x...%02x%02x\n", pHeader.signature_[0], pHeader.signature_[1], pHeader.signature_[DexFile::kSha1DigestSize - 2], pHeader.signature_[DexFile::kSha1DigestSize - 1]);
	fprintf(gOutFile, "file_size           : %d\n", pHeader.file_size_);
	fprintf(gOutFile, "header_size         : %d\n", pHeader.header_size_);
	fprintf(gOutFile, "link_size           : %d\n", pHeader.link_size_);
	fprintf(gOutFile, "link_off            : %d (0x%06x)\n", pHeader.link_off_, pHeader.link_off_);
	fprintf(gOutFile, "string_ids_size     : %d\n", pHeader.string_ids_size_);
	fprintf(gOutFile, "string_ids_off      : %d (0x%06x)\n", pHeader.string_ids_off_, pHeader.string_ids_off_);
	fprintf(gOutFile, "type_ids_size       : %d\n", pHeader.type_ids_size_);
	fprintf(gOutFile, "type_ids_off        : %d (0x%06x)\n", pHeader.type_ids_off_, pHeader.type_ids_off_);
	fprintf(gOutFile, "proto_ids_size      : %d\n", pHeader.proto_ids_size_);
	fprintf(gOutFile, "proto_ids_off       : %d (0x%06x)\n", pHeader.proto_ids_off_, pHeader.proto_ids_off_);
	fprintf(gOutFile, "field_ids_size      : %d\n", pHeader.field_ids_size_);
	fprintf(gOutFile, "field_ids_off       : %d (0x%06x)\n", pHeader.field_ids_off_, pHeader.field_ids_off_);
	fprintf(gOutFile, "method_ids_size     : %d\n", pHeader.method_ids_size_);
	fprintf(gOutFile, "method_ids_off      : %d (0x%06x)\n", pHeader.method_ids_off_, pHeader.method_ids_off_);
	fprintf(gOutFile, "class_defs_size     : %d\n", pHeader.class_defs_size_);
	fprintf(gOutFile, "class_defs_off      : %d (0x%06x)\n", pHeader.class_defs_off_, pHeader.class_defs_off_);
	fprintf(gOutFile, "data_size           : %d\n", pHeader.data_size_);
	fprintf(gOutFile, "data_off            : %d (0x%06x)\n\n", pHeader.data_off_, pHeader.data_off_);
}
```

下面两个表格出自 AI 大老爷，第一个是文件头

| 字段                 | 偏移（文件内字节） | 类型     | 描述                                             |
| ------------------ | --------- | ------ | ---------------------------------------------- |
| `magic_`           | 0x00      | u1[8]  | 魔数 + 版本号，例如 `"dex\n035\0"`，标识文件是 DEX           |
| `checksum_`        | 0x08      | u4     | Adler32 校验和，用于验证文件完整性（不包括 magic 和 checksum 自身） |
| `signature_`       | 0x0C      | u1[20] | SHA-1 签名，用于验证文件完整性                             |
| `file_size_`       | 0x20      | u4     | 文件总大小（字节）                                      |
| `header_size_`     | 0x24      | u4     | Header 大小，固定 0x70（112 字节）                      |
| `link_size_`       | 0x28      | u4     | 链接数据大小（旧 Dalvik 兼容，可忽略）                        |
| `link_off_`        | 0x2C      | u4     | 链接数据偏移                                         |
| `string_ids_size_` | 0x30      | u4     | 字符串索引表元素数量                                     |
| `string_ids_off_`  | 0x34      | u4     | 字符串索引表偏移                                       |
| `type_ids_size_`   | 0x38      | u4     | 类型索引表元素数量                                      |
| `type_ids_off_`    | 0x3C      | u4     | 类型索引表偏移                                        |
| `proto_ids_size_`  | 0x40      | u4     | 方法原型表元素数量                                      |
| `proto_ids_off_`   | 0x44      | u4     | 方法原型表偏移                                        |
| `field_ids_size_`  | 0x48      | u4     | 字段索引表元素数量                                      |
| `field_ids_off_`   | 0x4C      | u4     | 字段索引表偏移                                        |
| `method_ids_size_` | 0x50      | u4     | 方法索引表元素数量                                      |
| `method_ids_off_`  | 0x54      | u4     | 方法索引表偏移                                        |
| `class_defs_size_` | 0x58      | u4     | 类定义表元素数量                                       |
| `class_defs_off_`  | 0x5C      | u4     | 类定义表偏移                                         |
| `data_size_`       | 0x60      | u4     | 数据段总大小（字节）                                     |
| `data_off_`        | 0x64      | u4     | 数据段偏移（包含字符串、注解、类数据、调试信息等）                      |

这个是文件内容

| Section             | 偏移字段 (Header)    | 元素个数字段 (Header) | 描述                                                          | 备注              |
| ------------------- | -------------------- | --------------------- | ------------------------------------------------------------- | ----------------- |
| **文件头 (Header)** | —                    | —                     | 存储 DEX 文件的基本信息，包括魔数、版本、校验和、SHA-1 签名等 | 固定 0x70 字节    |
| **string_ids**      | `string_ids_off`     | `string_ids_size`     | 存储字符串索引表，每个元素指向 data section 中的 UTF-8 字符串 | 每个元素 4 字节   |
| **type_ids**        | `type_ids_off`       | `type_ids_size`       | 存储类型索引表，每个元素指向 string_ids                       | 每个元素 4 字节   |
| **proto_ids**       | `proto_ids_off`      | `proto_ids_size`      | 方法原型表，存储返回类型、参数类型索引等                      | 每个元素 12 字节  |
| **field_ids**       | `field_ids_off`      | `field_ids_size`      | 字段索引表，包含所属类、类型、字段名索引                      | 每个元素 8 字节   |
| **method_ids**      | `method_ids_off`     | `method_ids_size`     | 方法索引表，包含所属类、原型索引、方法名索引                  | 每个元素 8 字节   |
| **class_defs**      | `class_defs_off`     | `class_defs_size`     | 类定义表，描述类本身信息，包括字段、方法、接口、父类、注解    | 每个元素 32 字节  |
| **call_site_ids**   | `call_site_ids_off`  | `call_site_ids_size`  | 用于动态调用 `invoke-custom`                                  | 每个元素 4 字节   |
| **method_handles**  | `method_handles_off` | `method_handles_size` | 方法句柄，用于 lambda、动态调用                               | 每个元素 8 字节   |
| **link_data**       | `link_off`           | `link_size`           | 旧 Dalvik 兼容的链接信息（一般为 0）                          | 可忽略            |
| **data section**    | `data_off`           | `data_size`           | 存储实际字符串、编码的注解、类数据、调试信息、编码方法等      | 由各 section 引用 |

#### dumpclass 信息

继续看`processDexFile`中的核心函数`dumpClass`，这个函数负责输出一个类的完整信息：

```cpp
static void dumpClass(const DexFile *pDexFile, int idx, char **pLastPackage)
{
    const dex::ClassDef &pClassDef = pDexFile->GetClassDef(idx);

    // 1. 过滤非public类（如果设置了exportsOnly）
    if (gOptions.exportsOnly && (pClassDef.access_flags_ & kAccPublic) == 0) {
        return;
    }

    // 2. 输出类定义头部信息
    if (gOptions.showSectionHeaders) {
        dumpClassDef(pDexFile, idx);
    }

    // 3. 输出注解信息
    if (gOptions.showAnnotations) {
        dumpClassAnnotations(pDexFile, idx);
    }

    // 4. 输出控制流图（如果设置了showCfg）
    if (gOptions.showCfg) {
        dumpCfg(pDexFile, idx);
        return;  // 输出CFG后就不需要输出其他内容了
    }

    // 5. XML输出模式下的包管理
    const char *classDescriptor = pDexFile->StringByTypeIdx(pClassDef.class_idx_);
    if (!(classDescriptor[0] == 'L' &&
          classDescriptor[strlen(classDescriptor) - 1] == ';')) {
        // 不是标准类描述符（可能是数组或基本类型）
        LOG(WARNING) << "Malformed class name '" << classDescriptor << "'";
    } else if (gOptions.outputFormat == OUTPUT_XML) {
        // 处理包名，用于XML分组
        char *mangle = strdup(classDescriptor + 1);  // 去掉前面的'L'
        mangle[strlen(mangle) - 1] = '\0';           // 去掉后面的';'

        // 提取包名部分
        char *lastSlash = strrchr(mangle, '/');
        if (lastSlash != nullptr) {
            *lastSlash = '\0';  // 分隔符前是包名
        } else {
            *mangle = '\0';     // 默认包
        }

        // 将'/'替换为'.'
        for (char *cp = mangle; *cp != '\0'; cp++) {
            if (*cp == '/') *cp = '.';
        }

        // 如果包名变了，关闭旧的<package>标签，开启新的
        if (*pLastPackage == nullptr || strcmp(mangle, *pLastPackage) != 0) {
            if (*pLastPackage != nullptr) {
                fprintf(gOutFile, "</package>\n");
            }
            fprintf(gOutFile, "<package name=\"%s\"\n>\n", mangle);
            free(*pLastPackage);
            *pLastPackage = mangle;
        } else {
            free(mangle);
        }
    }

    // 6. 输出类的基本信息
    char *accessStr = createAccessFlagStr(pClassDef.access_flags_, kAccessForClass);
    const char *superclassDescriptor = pClassDef.superclass_idx_.IsValid()
        ? pDexFile->StringByTypeIdx(pClassDef.superclass_idx_)
        : nullptr;

    if (gOptions.outputFormat == OUTPUT_PLAIN) {
        // 普通文本输出
        fprintf(gOutFile, "Class #%d            -\n", idx);
        fprintf(gOutFile, "  Class descriptor  : '%s'\n", classDescriptor);
        fprintf(gOutFile, "  Access flags      : 0x%04x (%s)\n",
                pClassDef.access_flags_, accessStr);
        if (superclassDescriptor != nullptr) {
            fprintf(gOutFile, "  Superclass        : '%s'\n", superclassDescriptor);
        }
        fprintf(gOutFile, "  Interfaces        -\n");
    } else {
        // XML输出
        std::unique_ptr<char[]> dot(descriptorClassToName(classDescriptor));
        fprintf(gOutFile, "<class name=\"%s\"\n", dot.get());
        if (superclassDescriptor != nullptr) {
            dot = descriptorToDot(superclassDescriptor);
            fprintf(gOutFile, " extends=\"%s\"\n", dot.get());
        }
        fprintf(gOutFile, " interface=%s\n", quotedBool((pClassDef.access_flags_ & kAccInterface) != 0));
        fprintf(gOutFile, " abstract=%s\n", quotedBool((pClassDef.access_flags_ & kAccAbstract) != 0));
        fprintf(gOutFile, " static=%s\n", quotedBool((pClassDef.access_flags_ & kAccStatic) != 0));
        fprintf(gOutFile, " final=%s\n", quotedBool((pClassDef.access_flags_ & kAccFinal) != 0));
        fprintf(gOutFile, " visibility=%s\n", quotedVisibility(pClassDef.access_flags_));
        fprintf(gOutFile, ">\n");
    }

    // 7. 输出接口列表
    const dex::TypeList *pInterfaces = pDexFile->GetInterfacesList(pClassDef);
    if (pInterfaces != nullptr) {
        for (u4 i = 0; i < pInterfaces->Size(); i++) {
            dumpInterface(pDexFile, pInterfaces->GetTypeItem(i), i);
        }
    }

    // 8. 使用ClassAccessor访问类的字段和方法
    ClassAccessor accessor(*pDexFile, pClassDef, /* parse_hiddenapi_class_data= */ true);

    // 9. 准备静态字段的初始值数据
    const u1 *sData = pDexFile->GetEncodedStaticFieldValuesArray(pClassDef);
    const u4 sSize = sData != nullptr ? DecodeUnsignedLeb128(&sData) : 0;

    // 10. 输出静态字段
    if (gOptions.outputFormat == OUTPUT_PLAIN) {
        fprintf(gOutFile, "  Static fields     -\n");
    }
    uint32_t i = 0u;
    for (const ClassAccessor::Field &field : accessor.GetStaticFields()) {
        dumpField(field, i, i < sSize ? &sData : nullptr);
        ++i;
    }

    // 11. 输出实例字段
    if (gOptions.outputFormat == OUTPUT_PLAIN) {
        fprintf(gOutFile, "  Instance fields   -\n");
    }
    i = 0u;
    for (const ClassAccessor::Field &field : accessor.GetInstanceFields()) {
        dumpField(field, i);
        ++i;
    }

    // 12. 输出直接方法（构造函数、静态方法等）
    if (gOptions.outputFormat == OUTPUT_PLAIN) {
        fprintf(gOutFile, "  Direct methods    -\n");
    }
    i = 0u;
    for (const ClassAccessor::Method &method : accessor.GetDirectMethods()) {
        dumpMethod(method, i);
        ++i;
    }

    // 13. 输出虚方法
    if (gOptions.outputFormat == OUTPUT_PLAIN) {
        fprintf(gOutFile, "  Virtual methods   -\n");
    }
    i = 0u;
    for (const ClassAccessor::Method &method : accessor.GetVirtualMethods()) {
        dumpMethod(method, i);
        ++i;
    }

    // 14. 输出结束
    if (gOptions.outputFormat == OUTPUT_PLAIN) {
        // 输出源文件信息
        const char *fileName;
        if (pClassDef.source_file_idx_.IsValid()) {
            fileName = pDexFile->StringDataByIdx(pClassDef.source_file_idx_);
        } else {
            fileName = "unknown";
        }
        fprintf(gOutFile, "  source_file_idx   : %d (%s)\n\n",
                pClassDef.source_file_idx_.index_, fileName);
    } else if (gOptions.outputFormat == OUTPUT_XML) {
        fprintf(gOutFile, "</class>\n");
    }

    free(accessStr);
}
```

这个函数有几个关键点：

1. **ClassAccessor 的使用**：这是一个 C++辅助类，提供了访问类数据的便捷接口
2. **静态字段初始值**：通过`GetEncodedStaticFieldValuesArray`获取
3. **字段和方法的分类**：分为静态/实例字段、直接/虚方法

#### 字段和方法输出函数

```cpp
static void dumpField(const ClassAccessor::Field &field, int i, const u1 **data = nullptr)
{
    const uint32_t flags = field.GetAccessFlags();
    const DexFile &dex_file = field.GetDexFile();
    const dex::FieldId &field_id = dex_file.GetFieldId(field.GetIndex());
    const char *name = dex_file.StringDataByIdx(field_id.name_idx_);
    const char *typeDescriptor = dex_file.StringByTypeIdx(field_id.type_idx_);
    const char *backDescriptor = dex_file.StringByTypeIdx(field_id.class_idx_);
    char *accessStr = createAccessFlagStr(flags, kAccessForField);
    const uint32_t hiddenapiFlags = field.GetHiddenapiFlags();

    if (gOptions.outputFormat == OUTPUT_PLAIN) {
        fprintf(gOutFile, "    #%d              : (in %s)\n", i, backDescriptor);
        fprintf(gOutFile, "      name          : '%s'\n", name);
        fprintf(gOutFile, "      type          : '%s'\n", typeDescriptor);
        fprintf(gOutFile, "      access        : 0x%04x (%s)\n", flags, accessStr);
        if (hiddenapiFlags != 0u) {
            fprintf(gOutFile, "      hiddenapi     : 0x%04x (%s)\n",
                    hiddenapiFlags,
                    GetHiddenapiFlagStr(hiddenapiFlags).c_str());
        }
        if (data != nullptr) {
            fputs("      value         : ", gOutFile);
            dumpEncodedValue(&dex_file, data);
            fputs("\n", gOutFile);
        }
    } else if (gOptions.outputFormat == OUTPUT_XML) {
        // XML格式输出...
    }

    free(accessStr);
}
```

#### dumpMethod 函数

```cpp
static void dumpMethod(const ClassAccessor::Method &method, int i)
{
    const uint32_t flags = method.GetAccessFlags();
    const DexFile &dex_file = method.GetDexFile();
    const dex::MethodId &pMethodId = dex_file.GetMethodId(method.GetIndex());
    const char *name = dex_file.StringDataByIdx(pMethodId.name_idx_);
    const Signature signature = dex_file.GetMethodSignature(pMethodId);
    char *typeDescriptor = strdup(signature.ToString().c_str());
    const char *backDescriptor = dex_file.StringByTypeIdx(pMethodId.class_idx_);
    char *accessStr = createAccessFlagStr(flags, kAccessForMethod);
    const uint32_t hiddenapiFlags = method.GetHiddenapiFlags();

    if (gOptions.outputFormat == OUTPUT_PLAIN) {
        fprintf(gOutFile, "    #%d              : (in %s)\n", i, backDescriptor);
        fprintf(gOutFile, "      name          : '%s'\n", name);
        fprintf(gOutFile, "      type          : '%s'\n", typeDescriptor);
        fprintf(gOutFile, "      access        : 0x%04x (%s)\n", flags, accessStr);
        if (gOptions.showSectionHeaders) {
            fprintf(gOutFile, "      method_idx    : %d\n", method.GetIndex());
        }
        if (hiddenapiFlags != 0u) {
            fprintf(gOutFile,
                    "      hiddenapi     : 0x%04x (%s)\n",
                    hiddenapiFlags,
                    GetHiddenapiFlagStr(hiddenapiFlags).c_str());
        }
        if (method.GetCodeItem() == nullptr) {
            fprintf(gOutFile, "      code          : (none)\n");
        } else {
            fprintf(gOutFile, "      code          -\n");
            dumpCode(&dex_file,
                    method.GetIndex(),
                    flags,
                    method.GetCodeItem(),
                    method.GetCodeItemOffset());
        }
        if (gOptions.disassemble) {
            fputc('\n', gOutFile);
        }
    } else if (gOptions.outputFormat == OUTPUT_XML) {
        // XML格式输出...
    }

    free(typeDescriptor);
    free(accessStr);
}
```

#### 字节码反汇编

```cpp
static void dumpCode(const DexFile *pDexFile, u4 idx, u4 flags,
                     const dex::CodeItem *pCode, u4 codeOffset)
{
    CodeItemDebugInfoAccessor accessor(*pDexFile, pCode, idx);

    fprintf(gOutFile, "      registers     : %d\n", accessor.RegistersSize());
    fprintf(gOutFile, "      ins           : %d\n", accessor.InsSize());
    fprintf(gOutFile, "      outs          : %d\n", accessor.OutsSize());
    fprintf(gOutFile, "      insns size    : %d 16-bit code units\n",
            accessor.InsnsSizeInCodeUnits());

    // 如果需要反汇编，输出字节码
    if (gOptions.disassemble) {
        dumpBytecodes(pDexFile, idx, pCode, codeOffset);
    }

    // 异常处理表
    dumpCatches(pDexFile, pCode);

    // 调试信息：行号表和局部变量表
    bool is_static = (flags & kAccStatic) != 0;
    fprintf(gOutFile, "      positions     : \n");
    accessor.DecodeDebugPositionInfo([&](const DexFile::PositionInfo &entry) {
        fprintf(gOutFile, "        0x%04x line=%d\n", entry.address_, entry.line_);
        return false;
    });
    fprintf(gOutFile, "      locals        : \n");
    accessor.DecodeDebugLocalInfo(is_static,
                                  idx,
                                  [&](const DexFile::LocalInfo &entry) {
        const char *signature = entry.signature_ != nullptr ? entry.signature_ : "";
        fprintf(gOutFile,
                "        0x%04x - 0x%04x reg=%d %s %s %s\n",
                entry.start_address_,
                entry.end_address_,
                entry.reg_,
                entry.name_,
                entry.descriptor_,
                signature);
    });
}
```

#### `dumpBytecodes`函数

```cpp
static void dumpBytecodes(const DexFile *pDexFile, u4 idx,
                          const dex::CodeItem *pCode, u4 codeOffset)
{
    const dex::MethodId &pMethodId = pDexFile->GetMethodId(idx);
    const char *name = pDexFile->StringDataByIdx(pMethodId.name_idx_);
    const Signature signature = pDexFile->GetMethodSignature(pMethodId);
    const char *backDescriptor = pDexFile->StringByTypeIdx(pMethodId.class_idx_);

    // 生成方法头
    std::unique_ptr<char[]> dot(descriptorToDot(backDescriptor));
    fprintf(gOutFile, "%06x:                                        |[%06x] %s.%s:%s\n",
            codeOffset, codeOffset, dot.get(), name, signature.ToString().c_str());

    // 遍历所有指令
    CodeItemDataAccessor accessor(*pDexFile, pCode);
    const u4 maxPc = accessor.InsnsSizeInCodeUnits();
    for (const DexInstructionPcPair &pair : accessor) {
        const u4 dexPc = pair.DexPc();
        if (dexPc >= maxPc) {
            LOG(WARNING) << "GLITCH: run-away instruction at idx=0x" << std::hex << dexPc;
            break;
        }
        const Instruction *instruction = &pair.Inst();
        const u4 insnWidth = instruction->SizeInCodeUnits();
        if (insnWidth == 0) {
            LOG(WARNING) << "GLITCH: zero-width instruction at idx=0x" << std::hex << dexPc;
            break;
        }
        dumpInstruction(pDexFile, pCode, codeOffset, dexPc, insnWidth, instruction);
    }
}
```

### `dumpInstruction`函数

这个函数非常长，负责将 Dalvik/ART 字节码指令反汇编为可读文本。它处理各种指令格式：

- `k10x`: 无操作数指令
- `k12x`: 两个寄存器操作数
- `k21c`: 寄存器和索引操作数
- `k35c`: 调用指令，带多个寄存器参数  
   等等...
  指令反汇编的核心是`indexString`函数，它负责解析索引操作数（字段引用、方法引用、类型引用等）。

## 总结

通过分析`dexdump.cc`，我们可以学到：

1. **DEX 文件结构**：从头信息、字符串表、类型表、原型表、字段表、方法表到类定义表
2. **类结构**：访问标志、超类、接口、字段、方法、注解
3. **字节码反汇编**：如何将 Dalvik/ART 字节码转换为可读文本
4. **调试信息**：行号表、局部变量表
5. **新特性**：方法句柄和调用站点的支持
   `dexdump`工具实际上是一个完整的 DEX 文件解析器，它展示了 DEX 文件在内存中的完整结构。虽然代码量很大（近 3000 行），但结构清晰，按照 DEX 文件的物理结构组织代码。
   这种实现方式反映了 Android 系统对 DEX 文件的理解：它不仅仅是字节码的容器，还包含了完整的类型系统、调试信息和安全特性（如 hiddenapi 标志）。

说实话，我现在虽然知道这个有什么了，但是我本身还是懵逼的，我也不知道学了什么，我连我懵逼什么都不知道