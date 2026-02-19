# EC2 Access Manual

## 1) Prerequisites

- A terminal (Linux/macOS) or WSL/Git Bash (Windows)
- The private key file (PEM)
- Network access to the host and the published SSH ports

---

## 2) PEM file

Place the PEM in a known path, for example:

```bash
~/.ssh/deploytour
```

---

## 3) Key preparation (PEM)

### 3.1 Permissions

```bash
chmod 600 ~/.ssh/deploytour
```

---

## 4) SSH access

Access is done through the host **ptwall.iti.es** using different ports for each VM.

### 4.1 Access to deploytour-worker

- User: `ubuntu`
- Host: `ptwall.iti.es`
- Port: `50100`

```bash
ssh -i ~/.ssh/deploytour -p 50100 ubuntu@ptwall.iti.es
```

### 4.2 Access to deploytour-master

- User: `ubuntu`
- Host: `ptwall.iti.es`
- Port: `50101`

```bash
ssh -i ~/.ssh/deploytour -p 50101 ubuntu@ptwall.iti.es
```

---

## 5) Verification

Check identity and resources:

```bash
hostname
lsb_release -a
free -h
nproc
df -h
```

---

## 6) Upload loose files with scp (quick and simple)

### 6.1 Upload a file to master

```bash
scp -i ~/.ssh/deploytour -P 50101 ./mi_archivo.zip ubuntu@ptwall.iti.es:
```

### 6.2 Upload a folder/project to master

```bash
scp -i ~/.ssh/deploytour -P 50101 -r ./mi_proyecto ubuntu@ptwall.iti.es:
```

### 6.3 Same but to worker

```bash
scp -i ~/.ssh/deploytour -P 50100 -r ./mi_proyecto ubuntu@ptwall.iti.es:
```
