
```bash
source setvars.sh

ZES_ENABLE_SYSMAN=1 \
LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$(pwd)/Tools/llama.cpp/build/bin \
Tools/llama.cpp/build/bin/llama-cli \
-m Tools/ai-models/qwen2.5-coder-7b-instruct-q4_k_m.gguf \
-ngl 32 \
--device sycl0 \
-p "coba buat halaman web untuk marketplace dengan gaya modern dan minimalis"
```


.bashrc PATH 

```bash
export LLAMA=~/Tools/llama.cpp/build/bin
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$LLAMA
```

```bash
ZES_ENABLE_SYSMAN=1 $LLAMA/llama-server -m ~/Tools/ai-models/Qwen2.5-Coder-3B-Instruct-Q8_0.gguf -ngl 99 --device sycl0
```
