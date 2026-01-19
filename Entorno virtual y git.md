# 0) Pre-requisitos (una vez)

```bash
# Ver versión de Python
python --version

# (Opcional) Configurar identidad global de Git
git config --global user.name "Tu Nombre"
git config --global user.email "tuemail@dominio.com"

# (Opcional) Alias útiles
git config --global alias.st "status"
git config --global alias.hist "log --graph --decorate --all --oneline"
```

---

# 1) Crear proyecto base

```bash
# Crea carpeta y entra
mkdir mi_proyecto && cd mi_proyecto

# Estructura mínima
mkdir -p src tests
echo "# Mi Proyecto" > README.md
echo "__pycache__/\n*.pyc\n.venv/\nvenv/\n.ipynb_checkpoints/\n.env\n.DS_Store\n*.log\nuv.lock\n" > .gitignore
```

---

# 2A) Camino A — pip + venv (clásico)

## 2A.1 Crear y activar entorno

```bash
# Windows
python -m venv .venv
.\.venv\Scripts\activate

# macOS / Linux
python -m venv .venv
source .venv/bin/activate
```

## 2A.2 Instalar dependencias

```bash
pip install --upgrade pip
pip install pandas matplotlib ipykernel
```

## 2A.3 Exportar requirements

```bash
pip freeze > requirements.txt
```

## 2A.4 Registrar kernel para Jupyter/VS Code

```bash
python -m ipykernel install --user --name mi_proyecto --display-name "mi_proyecto (venv)"
```

> En VS Code: abre el `.ipynb` → selecciona kernel **“mi\_proyecto (venv)”**.

---

# 2B) Camino B — uv (moderno, súper rápido)

## 2B.1 Inicializar proyecto y Python

```bash
uv init --python 3.12  # o tu versión
# (si ya tienes carpeta, ejecuta dentro: uv init)
```

## 2B.2 Añadir dependencias

```bash
uv add pandas matplotlib ipykernel
```

## 2B.3 Sincronizar (instala lo del pyproject)

```bash
uv sync
```

## 2B.4 Usar el entorno / ejecutar

```bash
uv shell             # activa entorno
uv run python -V     # ejecuta con el entorno
```

## 2B.5 Kernel de Jupyter

```bash
uv run python -m ipykernel install --user --name mi_proyecto --display-name "mi_proyecto (uv)"
```

---

# 3) Inicializar Git en local

```bash
git init
git add .
git commit -m "chore: bootstrap proyecto (estructura, entorno y deps)"
```

> Si ya tenías archivos sin querer en staging: `git reset` (los baja del staging sin perder cambios).

---

# 4) Crear el repo en GitHub y vincular remoto

## Opción HTTPS

```bash
git remote add origin https://github.com/tu_usuario/mi_proyecto.git
git branch -M main          # usa 'main' como rama principal
git push -u origin main
```

## Opción SSH (recomendado si usas claves)

```bash
# (Primera vez) Genera clave y súbela a GitHub (Settings > SSH and GPG keys)
ssh-keygen -t ed25519 -C "tuemail@dominio.com"
# copia la clave pública (~/.ssh/id_ed25519.pub) a GitHub

git remote add origin git@github.com:tu_usuario/mi_proyecto.git
git branch -M main
git push -u origin main
```

---

# 5) Flujo de trabajo recomendado (GitHub Flow)

## 5.1 Crear rama de feature

```bash
git switch -c feat/analitica   # o: git checkout -b feat/analitica
```

## 5.2 Trabajar y commitear

```bash
git status
git add .
git commit -m "feat: añade módulo de analítica inicial"
```

## 5.3 Mantenerte al día con main

```bash
git fetch origin
git switch main
git pull --rebase origin main
git switch feat/analitica
git rebase main               # o merge si lo prefieres
```

## 5.4 Subir tu rama y abrir Pull Request

```bash
git push -u origin feat/analitica
# Abre el PR en GitHub, revisa, aprueba y mergea a main
```

## 5.5 Taggear versión (opcional)

```bash
git tag -a v0.1.0 -m "Primera versión"
git push origin v0.1.0
```

---

# 6) Sincronizar dependencias entre máquinas/compañeros

**pip+venv**

```bash
# Clonar y preparar
git clone https://github.com/tu_usuario/mi_proyecto.git
cd mi_proyecto
python -m venv .venv
# activar (Windows) .\.venv\Scripts\activate  |  (Unix) source .venv/bin/activate
pip install -r requirements.txt
```

**uv**

```bash
git clone https://github.com/tu_usuario/mi_proyecto.git
cd mi_proyecto
uv sync
```

---

# 7) Jupyter en VS Code (recordatorio rápido)

1. Abre el `.ipynb`
2. Selector de kernel (arriba a la derecha)
3. Escoge **“mi\_proyecto (venv)”** o **“mi\_proyecto (uv)”**

---

# 8) Comandos Git de referencia rápida

```bash
# Estado / historial
git st
git hist
git log --oneline
git log --graph --decorate --all

# Diferencias
git diff                 # cambios sin staged
git diff --staged        # staged vs HEAD
git diff HASH1 HASH2     # entre commits

# Recuperación
git reflog               # movimientos de HEAD
git reset archivo.py     # saca del staging
git reset --soft HASH    # mueve HEAD, conserva staged
git reset --hard HASH    # vuelve al commit exacto (¡destructivo!)
git reset --hard HEAD    # descarta cambios locales

# Ramas modernas
git switch main
git switch -c fix/bug-x

# Remoto
git remote -v
git pull --rebase origin main
git push -u origin rama
```

---

# 9) Extras recomendados

```bash
# Licencia y plantillas (opcional)
echo "MIT License ..." > LICENSE
mkdir -p .github
echo "Plantilla de PR" > .github/pull_request_template.md

# Pre-commit hooks (lint/format antes del commit)
pip install pre-commit         # o: uv add --dev pre-commit
pre-commit install
```

---

## 🧭 Resumen mínimo (si tienes prisa)

**pip+venv**

```bash
python -m venv .venv && source .venv/bin/activate
pip install -U pip pandas matplotlib ipykernel
pip freeze > requirements.txt
python -m ipykernel install --user --name mi_proyecto --display-name "mi_proyecto (venv)"
git init && git add . && git commit -m "init"
git branch -M main
git remote add origin <https/ssh>
git push -u origin main
```

**uv**

```bash
uv init --python 3.12
uv add pandas matplotlib ipykernel
uv sync
uv run python -m ipykernel install --user --name mi_proyecto --display-name "mi_proyecto (uv)"
git init && git add . && git commit -m "init"
git branch -M main
git remote add origin <https/ssh>
git push -u origin main
```

Si quieres, te lo preparo como **plantilla de README** o un **script `.ps1` / `.sh`** para automatizar todo (elige pip+venv o uv).
