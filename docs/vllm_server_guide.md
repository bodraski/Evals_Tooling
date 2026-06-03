# Гайд по работе с vLLM сервером
 
## Подключение по SSH
 
### Первый раз
 
Добавь в `~/.ssh/config` на своей машине:
 
```
Host vllm-server
    HostName <IP сервера>
    User <твой_логин>
    IdentityFile ~/.ssh/id_ed25519
    ServerAliveInterval 30
    ServerAliveCountMax 6
    TCPKeepAlive yes
    ControlMaster auto
    ControlPersist 1h
```
 
Подключение:
```bash
ssh vllm-server
```
 
---
 
## Обязательно: tmux
 
Все запуски vLLM делаются только внутри tmux-сессии. Если запустить без tmux — при обрыве SSH контейнер упадёт.
 
```bash
# создать новую сессию
tmux new -s vllm-session
 
# переподключиться к существующей (если уже запускал раньше)
tmux attach -t vllm-session
 
# отключиться не убивая сессию (контейнер продолжает работать)
# Ctrl+B, затем D
 
# посмотреть список активных сессий
tmux ls
```
 
---
 
## Запуск vLLM
 
### Базовый запуск (дефолтная модель, дефолтные параметры)
 
```bash
sudo /opt/vllm/run.sh
```
 
### С параметрами
 
```bash
# другая модель
sudo /opt/vllm/run.sh --model solidrust/Mistral-7B-Instruct-v0.3-AWQ
 
# изменить длину контекста (максимум 8192)
sudo /opt/vllm/run.sh --max-model-len 2048
 
# комбинация
sudo /opt/vllm/run.sh --model Qwen/Qwen2.5-7B-Instruct-AWQ --max-model-len 4096
 
# дополнительные параметры vLLM
sudo /opt/vllm/run.sh --max-model-len 4096 --gpu-memory-utilization 0.95
```
 
### Доступные модели
 
| Модель | Квантизация | Reasoning parser |
|--------|-------------|-----------------|
| `casperhansen/deepseek-r1-distill-qwen-7b-awq` | awq_marlin | deepseek_r1 |
| `Qwen/Qwen2.5-7B-Instruct-AWQ` | awq_marlin | — |
| `solidrust/Mistral-7B-Instruct-v0.3-AWQ` | awq_marlin | — |
 
Квантизация и reasoning parser подставляются автоматически по модели.
 
### Что нельзя передавать
 
Следующие параметры задаются скриптом автоматически, передавать их вручную нельзя:
- `--api-key`
- `--quantization`
- `--reasoning-parser`
### Остановка
 
`Ctrl+C` внутри tmux-сессии — контейнер остановится корректно, ресурсы освободятся.
 
---
 
## Проверка статуса
 
```bash
# кто сейчас занял сервер и с какими параметрами
cat /var/vllm/lock
 
# история всех запусков
cat /var/vllm/history.log
 
# логи контейнера (разово)
sudo docker logs vllm
 
# логи в реальном времени
sudo docker logs vllm -f
```
 
Если lock-файл есть, а контейнер не отвечает — скрипт при следующем запуске автоматически почистит устаревший lock.
 
---
 
## Запросы к API
 
Сервер поднимает OpenAI-совместимый API на порту `8765`.
 
```bash
# список моделей
curl http://<IP>:8765/v1/models \
  -H "Authorization: Bearer <api_key>"
 
# chat completion
curl http://<IP>:8765/v1/chat/completions \
  -H "Authorization: Bearer <api_key>" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "casperhansen/deepseek-r1-distill-qwen-7b-awq",
    "messages": [{"role": "user", "content": "привет"}],
    "max_tokens": 512,
    "temperature": 0.7
  }'
```
 
Параметры запроса (`temperature`, `max_tokens`, `top_p`) передаются в теле запроса и не зависят от запуска контейнера.
 
---
 
## Частые ситуации
 
**Сервер занят**
```
❌ vLLM сейчас занят:
user: colleague1 | started: 2026-05-26 10:00 | ...
```
Напиши в чат тому кто запустил.
 
**Запуск без tmux**
```
❌ Запускай внутри tmux-сессии
```
Сначала `tmux new -s vllm-session`, потом запускай скрипт.
 
**Модель не из списка**
```
❌ Модель 'xxx' не разрешена.
```
Используй только модели из таблицы выше. Добавить новую — через Яну.
 
**Мало места на диске**
```
❌ Мало места на диске
```
Сообщи Яне.
 
