STEP 1: Windows → WSL

Check WSL distributions:

wsl -l -v


Enter WSL:

wsl

STEP 2: GPU Check Inside WSL

Verify GPU visibility:

nvidia-smi


✅ GPU visible = OK

STEP 3: Docker + GPU Verification

Basic Docker checks:

docker
docker -v
docker images
docker ps


✅ GPU visible inside Docker = Docker GPU ready

STEP 4: Start Axolotl Docker Container

Run Axolotl container:

docker run --gpus all --rm -it axolotlai/axolotl:main-latest


Expected prompt:

root@container-id:/workspace/axolotl#

Explore Axolotl Codebase
pwd
ls


Navigate source:

cd src/axolotl
ls
sed -n '1,200p' train.py


CLI inspection:

cd cli
ls
sed -n '1,200p' main.py


Configs exploration:

cd /workspace/axolotl
ls examples
ls examples/llama-2
sed -n '1,200p' examples/llama-2/qlora.yaml

Create Working Directory
mkdir -p /workspace/my_runs
cd /workspace/my_runs

Create Training Config

Open editor:

nano sft_test.yaml


Nano controls:

Shift + Enter

Ctrl + O → Save

Ctrl + X → Exit

Paste exactly:

base_model: Qwen/Qwen2.5-7B-Instruct
tokenizer_type: AutoTokenizer

datasets:
  - path: timdettmers/openassistant-guanaco
    type: completion
    field: text
    
load_in_4bit: true
adapter: qlora

sequence_len: 2048
micro_batch_size: 1
gradient_accumulation_steps: 8
num_epochs: 1

learning_rate: 2e-4
optimizer: paged_adamw_8bit
lr_scheduler: cosine

fp16: true
gradient_checkpointing: true

output_dir: /workspace/my_runs/output_sft
logging_steps: 10
save_steps: 500


Save:

CTRL + O → Enter → CTRL + X

Start Training
axolotl train sft_test.yaml


Monitor GPU:

nvidia-smi -l 1


Check outputs:

ls -lh /workspace/my_runs/output_sft


Help commands:

axolotl --help
axolotl train --help

STEP 5: Fetch Example Configs
axolotl fetch examples

STEP 6: First Safe & Fast Training
axolotl train examples/llama-3/lora-1b.yml


⏱️ Time: ~15–30 minutes (GPU dependent)

STEP 7: Inference (Test Model)

CLI inference:

axolotl inference examples/llama-3/lora-1b.yml \
  --lora-model-dir ./outputs/lora-out


Optional Gradio UI:

axolotl inference examples/llama-3/lora-1b.yml \
  --lora-model-dir ./outputs/lora-out \
  --gradio

STEP 8: (Optional) Merge LoRA
axolotl merge-lora examples/llama-3/lora-1b.yml \
  --lora-model-dir ./outputs/lora-out


Final merged model location:

outputs/lora-out/merged/

🧹 Safe Cleanup Commands
Remove prepared datasets
rm -rf last_run_prepared/
rm -rf /workspace/my_runs/output_sft/checkpoint-*

Delete HuggingFace cache (models + datasets)
rm -rf ~/.cache/huggingface/

Delete Axolotl outputs
rm -rf /workspace/my_runs/output_sft/

Disk usage check (inside container)
df -h
du -sh ~/.cache/*

🐳 Docker Cleanup (Host Machine)

Remove everything unused:

docker system prune -a


Remove dangling images:

docker image prune


Remove stopped containers:

docker container prune


Remove unused volumes:

docker volume prune

🔄 Fresh Axolotl Container Start
docker run --gpus all -it --rm \
  -v $(pwd):/workspace \
  axolotlai/axolotl:main-latest

📦 Optional: Custom HuggingFace Cache Path
export HF_HOME=/mnt/data/hf_cache
export TRANSFORMERS_CACHE=/mnt/data/hf_cache

# python script for inferencing

from transformers import AutoModelForCausalLM, AutoTokenizer
from peft import PeftModel
import torch

base_model_id = "Qwen/Qwen2.5-7B-Instruct"
lora_path = "/workspace/my_runs/output_sft"

tokenizer = AutoTokenizer.from_pretrained(base_model_id)

base_model = AutoModelForCausalLM.from_pretrained(
    base_model_id,
    device_map="auto",
    torch_dtype=torch.float16
)

model = PeftModel.from_pretrained(base_model, lora_path)

prompt = "Explain QLoRA in simple words"
inputs = tokenizer(prompt, return_tensors="pt").to(model.device)

outputs = model.generate(
    **inputs,
    max_new_tokens=200,
    temperature=0.7
)

print(tokenizer.decode(outputs[0], skip_special_tokens=True))

✅ CASE 2: Merged model (LoRA + Base already merged)

Agar tune ye command chalayi ho:

axolotl merge-lora sft_test.yaml \
  --lora-model-dir /workspace/my_runs/output_sft


Toh merged model yahan milega:

/workspace/my_runs/output_sft/merged/

🔹 Python inference (merged model – simplest)
from transformers import AutoModelForCausalLM, AutoTokenizer
import torch

model_path = "/workspace/my_runs/output_sft/merged"

tokenizer = AutoTokenizer.from_pretrained(model_path)
model = AutoModelForCausalLM.from_pretrained(
    model_path,
    device_map="auto",
    torch_dtype=torch.float16
)

prompt = "Explain QLoRA in simple words"
inputs = tokenizer(prompt, return_tensors="pt").to(model.device)

outputs = model.generate(**inputs, max_new_tokens=200)

print(tokenizer.decode(outputs[0], skip_special_tokens=True))

