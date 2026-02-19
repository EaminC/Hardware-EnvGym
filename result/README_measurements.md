# Actual measurement (no estimates)

The result JSONs are updated from **real process-tree measurements** only when you run the measurement scripts on your machine. This environment cannot execute long-running clone/build/test or write files back to the workspace.

## One command: run all measurements and update JSONs

From the **Label** directory (or pass its path):

```bash
bash /path/to/Label/run_all_measurements.sh
```

This will:

1. **grpc_grpc-go**: clone grpc-go → `go build ./...` + `go test -short ./...` with process-tree RSS sampling → write `result/grpc_grpc-go_peaks.txt` → update `result/grpc_grpc-go.json`
2. **gluetest**: clone gluetest → Maven compile + test (commons-csv) with process-tree RSS → `result/gluetest_peaks.txt` → `result/gluetest.json`
3. **facebook_zstd**: clone zstd → `make` + `make check` with process-tree RSS → `result/facebook_zstd_peaks.txt` → `result/facebook_zstd.json`
4. **fmtlib_fmt**: clone fmt → cmake build + ctest with process-tree RSS → `result/fmtlib_fmt_peaks.txt` → `result/fmtlib_fmt.json`
5. **flex**: clone flex → `pip install -r requirements.txt` + minimal run with process-tree RSS → `result/flex_peaks.txt` → `result/flex.json`

Requirements: `git`, and per-repo: **go** (grpc), **mvn** (gluetest), **make** (zstd), **cmake/ctest** (fmt), **pip/python3** (flex). Each measure script uses `measure_common.sh` to sample the **whole process tree** RSS every 0.5s and take the maximum (no single-process undercount).

## Run a single repo

```bash
bash /path/to/Label/measure_grpc_ram.sh
bash /path/to/Label/update_grpc_result_from_peaks.sh
```

```bash
bash /path/to/Label/measure_facebook_zstd.sh
python3 /path/to/Label/update_result_from_peaks.py /path/to/Label/result/facebook_zstd_peaks.txt /path/to/Label/result/facebook_zstd.json
```

(Same pattern for gluetest, fmtlib_fmt, flex with their `measure_*.sh` and corresponding `*_peaks.txt` / `*.json`.)
