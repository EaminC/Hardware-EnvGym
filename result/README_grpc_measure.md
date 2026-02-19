# grpc-go 最小硬件测量说明

结果文件 `grpc_grpc-go.json` 中的 **peaks_observed** 与 **min_hardware** 需来自**实测**（进程树总 RSS），而不是单进程 `time -v`。

## 如何得到“测量出来的”结果

1. **测量峰值**（约 2–3 分钟，需已安装 Go 和 git）  
   在**本机**执行：
   ```bash
   bash /home/cc/Label/measure_grpc_ram.sh
   ```
   - 会自动克隆 `grpc-go` 到 `Label/grpc-go-measure`（若不存在）
   - 运行 `go build ./...` 与 `go test -short ./...`，每 0.5 秒对**整棵进程树**的 RSS 求和并取最大值
   - 输出写入 `Label/result/grpc_grpc-go_peaks.txt`，内容形如：
     ```
     build_peak_rss_kb=...
     test_peak_rss_kb=...
     peak_disk_kb=...
     ```

2. **用实测值更新 JSON**  
   再执行：
   ```bash
   bash /home/cc/Label/update_grpc_result_from_peaks.sh
   ```
   - 读取 `grpc_grpc-go_peaks.txt`
   - 按 tutorial 边际（RAM×1.3+1 GB、Disk×1.3）计算 min_ram_gb、min_disk_gb
   - 写回 `result/grpc_grpc-go.json` 的 `peaks_observed` 与 `min_hardware`

完成后，`grpc_grpc-go.json` 中的数值即为**进程树实测**结果。
