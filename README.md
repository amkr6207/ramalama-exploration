# [Outreachy 2026] RamaLama exploration on Linux Mint 22

## Summary
Initially I planned to solve this assignment in two ways:
1. Inside a Podman container
2. On host using Python `venv` + Podman

I completed both explorations and documented commands, outputs, failures, and reasoning.

---

## System setup
- OS: Linux Mint 22.3
- RAM: 8 GB
- CPU: Intel i5 (8th gen)
- Storage: 512 GB SSD
- Container engine: Podman
- RamaLama: 0.18.0

---

## Approach 1: Running inside Podman container (initial attempt)

I first tried running RamaLama inside a generic Python container.

```bash
sudo apt install podman -y
```
![Podman installation](images/01-install-podman.png)

```bash
podman --version
```
![Podman version](images/02-podman-version.png)

```bash
podman pull docker.io/library/python:3.12
```
![Pull Python container image](images/03-podman-pull-python-3.12..png)

```bash
podman run -it python:3.12 bash
```

```bash
pip install ramalama
```

```bash
ramalama version
```

Then I tried pulling/running an Ollama model:

```bash
ramalama pull ollama://gemma:2b
ramalama run ollama://gemma:2b "What are the Four Foundations of the Fedora project?"
```

Observed issue:
- `Error: [Errno 2] No such file or directory: 'llama-server'`

Analysis:
- Running RamaLama inside a generic Python container was not reliable for my setup.
- I moved to host-based execution (next approach).

---

## Approach 2: Host + venv + Podman (main working flow)

I switched to host terminal and used `venv` environment with Podman available.

### Version check
```bash
ramalama version
```

Output:
- `ramalama version 0.18.0`

---

## Transport 1: Ollama (`ollama://`)

### Pull
```bash
ramalama pull ollama://gemma:2b
```

Output:
- `Using cached ollama://library/gemma:2b ...` (after successful download)

### Run
```bash
ramalama run ollama://gemma:2b "What are the Four Foundations of the Fedora project?"
```

Output (model answer was incorrect):
- Returned unrelated foundations like technical stability/security/scalability/user experience.

I also tested:

```bash
ramalama run ollama://llama3.2:3b "What are the Four Foundations of the Fedora project? Give only the official four names."
```

Output:
- `I cannot verify the names of the official four foundations of the Fedora project.`

And:

```bash
ramalama run --temp 0 ollama://granite3.1-dense:2b "Give only the official four Fedora Foundations names."
```

Output:
- Still incorrect.

Finding:
- Ollama transport worked technically, but small local models were inconsistent on this Fedora-specific factual question.

---

## Transport 2: Hugging Face (`huggingface://`)

### Pull
```bash
ramalama pull huggingface://Qwen/Qwen2.5-3B-Instruct-GGUF/qwen2.5-3b-instruct-q4_k_m.gguf
```

### Run (open prompt)
```bash
ramalama run --temp 0 huggingface://Qwen/Qwen2.5-3B-Instruct-GGUF/qwen2.5-3b-instruct-q4_k_m.gguf "Give only the official four Fedora Foundations names."
```

Output:
- Incorrect names.

### Run (constrained prompt)
```bash
ramalama run --temp 0 huggingface://Qwen/Qwen2.5-3B-Instruct-GGUF/qwen2.5-3b-instruct-q4_k_m.gguf "What are Fedora's Four Foundations? Answer exactly as: Freedom, Friends, Features, First."
```

Output:
- `Freedom, Friends, Features, First.`

Finding:
- Hugging Face transport worked.
- Prompt design significantly changed factual output quality.

---

## Transport 3: OCI Registry (`oci://`)

I tested multiple OCI registry references:

```bash
ramalama pull oci://quay.io/ramalama/tinyllama
ramalama pull oci://quay.io/rhatdan/tiny:latest
ramalama pull oci://quay.io/mmortari/gguf-py-example/v1/example.gguf
```

Outputs:
- All returned `does not exist` in my environment.

Then I tried local OCI conversion:

```bash
ramalama convert ollama://gemma:2b oci://gemma2b:latest
```

Output:
- Manifest creation error:
  - `invalid repository name ... cannot specify 64-byte hexadecimal strings`
  - `Failed to create manifest ...`
  - `Tagging build instead`

I confirmed a local image existed:

```bash
podman images | grep -Ei "gemma2b|smollm|oci"
```

Output:
- `localhost/gemma2b latest ...`

Then tried running:

```bash
ramalama run oci://localhost/gemma2b:latest "What are the Four Foundations of the Fedora project? Give only the official four names."
```

Output:
- `Error: subpath: invalid mount option`

Finding:
- OCI transport testing revealed real registry availability + runtime/mount issues in my stack.
- This was the most problematic transport on my setup.

---

## Additional debugging finding (`ramalama list`)

`ramalama list` crashed with:

- `AttributeError: 'str' object has no attribute 'timestamp'`

Workaround:

```bash
ramalama --nocontainer list
```

This worked and listed my local Ollama/HF models.

---

## Comparison and evaluation

1. Ollama transport:
- Easy to use.
- Worked technically.
- Factual correctness was inconsistent across tested models.

2. Hugging Face transport:
- Also easy and reliable in my environment.
- With better prompt constraints, returned correct Fedora foundations.

3. OCI transport:
- Most difficult in my setup.
- Public OCI references tested were unavailable.
- Local OCI conversion/run exposed additional errors.

4. Model behavior:
- Bigger parameter count did not guarantee correct answer.
- Prompt structure (`--temp 0` + constrained answer format) improved results significantly.

5. Official expected answer used for evaluation:
- `Freedom, Friends, Features, First`

---

## Does RamaLama make working with AI "boring"?

In my opinion: yes, in a good way for workflow consistency.
The same CLI flow (`pull`, `run`, transport prefixes) makes experimentation simple.
At the same time, model quality and runtime edge cases are still real, so debugging skill is still required.
