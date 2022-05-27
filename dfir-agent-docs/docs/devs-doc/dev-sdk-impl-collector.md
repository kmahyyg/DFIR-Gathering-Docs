---
sidebar_position: 3
---

# SDK 文档 - 收集器

## 接口定义

```go
type Collector interface {
	Run(result chan CollectOutput, errData chan WrappedError)
	Register() error // factory method
	Description() (desc string)
	Tags() (tags []string)
	IsConfigRequired() bool // if required, you must implement SetConfig and GetConfig
	Version() (version int)
	Name() (pluginName string)
	DependsOn() (deps []string)
}
```

实现上述接口即可。其他参见注释。

## 示例插件：

若收集的属于运行指标，需额外 PR 一个 proto3 格式的数据序列化 Protobuf 定义，用于传输和存储数据供分析使用。若收集的属于文件类型，请根据实际情况选择存储方式并联系作者提供相关读写配置文件的帮助。

Protobuf 中定义的消息结构节选如下：
```protobuf
syntax = "proto3";

package os_info;

// current working directory-based
option go_package = "./pkg/pbuf";

message HostInfoHardware {
  string release_version = 1;
  string kernel_version = 2;
  string additional_sysinfo = 3;
  repeated DiskInfo disk_info = 6;
  string hostname = 7;
}

message DiskInfo {
  string disk_name = 1;
  string disk_mount_point = 2;
  string disk_fstype = 3;
  string disk_mount_opts = 4;
  uint64 disk_total_size = 5;
  uint64 disk_free_size = 6;
  int32 disk_used_percent = 7;
  int32 disk_inode_used_percent = 8;
}
```


Go 的代码结构应当如下所示，实际完成底层运行的函数可以另写一个文件存储（便于维护和分系统处理），或写在当前文件内：

```go
package os_info

import (
	"github.com/kmahyyg/DFIR-Gathering/pkg/abstract"
	"github.com/kmahyyg/DFIR-Gathering/pkg/logging"
	"github.com/kmahyyg/DFIR-Gathering/pkg/pbuf"
	"github.com/sirupsen/logrus"
	"google.golang.org/protobuf/proto"
)

const (
	// those constants must be existing in formatted form for further compliance
	// plugin id issued by official, leave it empty or zero
	C_HostInfo_ID = 1
	// plugin name and output prefix
	C_HostInfo_Prefix = "hostinfo"
	// for updating and config version matching
	C_HostInfo_Ver = 1
)

var (
	// those variables must be existing using the same form to make sure restriction working
	_ abstract.Collector = (*HostInfoCollector)(nil)
	// situational task assign
	C_HostInfo_Tags = []string{"host", "required"}
	// running precedence
	C_HostInfo_Depends = []string{}
)

type HostInfoCollector struct {
	// logger is required with a prefix
	Logger *logrus.Entry
	// other fields could be added as you pleased
}

// MUST use pointer receiver for run
func (h *HostInfoCollector) Run(result chan abstract.CollectOutput, errData chan abstract.WrappedError) {
	// this function will be run asynchronously, make sure inside is blocking API.
	// actual work progress
	hardwareInfo := &pbuf.HostInfoHardware{}
	// real collect progress
	var err error
	hardwareInfo.CpuInfo, err = h.collectCPUInfo()
	if err != nil {
		errData <- abstract.WrappedError{
			Origin:      "run-cpuinfo",
			PluginName:  C_HostInfo_Prefix,
			ErrOriginal: err,
		}
	}
	hardwareInfo.DiskInfo, err = h.collectDiskInfo()
	if err != nil {
		errData <- abstract.WrappedError{
			Origin:      "run-diskinfo",
			PluginName:  C_HostInfo_Prefix,
			ErrOriginal: err,
		}
	}
	sysRelInfo, err := h.collectOSInfo()
	if err != nil {
		errData <- abstract.WrappedError{
			Origin:      "run-osinfo",
			PluginName:  C_HostInfo_Prefix,
			ErrOriginal: err,
		}
	}
	// extract info from result
	hardwareInfo.ReleaseVersion = sysRelInfo.ReleaseVersion
	hardwareInfo.KernelVersion = sysRelInfo.KernelVersion
	hardwareInfo.AdditionalSysinfo = sysRelInfo.AdditionalSysinfo
	hardwareInfo.Hostname = sysRelInfo.Hostname
	// packed into CollectOutput structure and send it out.
	bytesData, err := proto.Marshal(hardwareInfo)
	// error handling first
	if err != nil {
		// error should be wrapped and send out
		errData <- abstract.WrappedError{
			// origin should be the caller function name, and brief explain when it dies
			Origin:      "run-marshal-protobuf",
			PluginName:  C_HostInfo_Prefix,
			ErrOriginal: err,
		}
		// early return instead of if-else statement, do NOT use panic here
		return
	}
	// pack then go
	result <- abstract.CollectOutput{
		// marshalled protobuf data
		OutputBytes: bytesData,
		// for non-file collection, use <SOURCE>-metrics as location
		OriginalLogLocation: "sys-metrics",
		// per here, must use prefix defined above, DO NOT change it.
		PluginName: C_HostInfo_Prefix,
		// if there any sub-plugin, use Prefix-"sub_plugin_name" here
		OutputPrefix: C_HostInfo_Prefix,
	}
	// log finish then return
	h.Logger.Info("HostInfo collector successfully finished.")
}

// MUST use pointer receiver for register
func (h *HostInfoCollector) Register() error {
	// initialize logger here, this function will be guaranteed to be called at very first.
	h.Logger = logging.NewLogger(C_HostInfo_Prefix)
	// if need to initialize or read config from file, do it here.
	return nil
}

func (h HostInfoCollector) Description() (desc string) {
	// for showing description
	desc = "Acquiring Host Basic Information for Hardware and OS related."
	return
}

func (h HostInfoCollector) Tags() (tags []string) {
	// for showing tags, this will help us build a situational gathering task list
	return C_HostInfo_Tags
}

func (h HostInfoCollector) Version() (version int) {
	// return for version
	return C_HostInfo_Ver
}

func (h HostInfoCollector) Name() (pluginName string) {
	// return name for identification and further detection and processing
	return C_HostInfo_Prefix
}

func (h HostInfoCollector) IsConfigRequired() bool {
	// if return true, it will call GetConfig() and SetConfig() for corresponding purpose
	// 			and SetConfig must be called ONLY on pointer receiver
	// if return false, no action
	return false
}

func (h HostInfoCollector) DependsOn() (deps []string) {
	// this will add precedence for collector and detector to be called
	// all collectors will be executed in an ordered queue with priority
	// note: it is your responsibility to check document before set it
	//       do not make it in a dead loop, retry 2 times to reload will
	// 		 make program mark it as incomplete plugin and failed to run
	return C_HostInfo_Depends
}
```
