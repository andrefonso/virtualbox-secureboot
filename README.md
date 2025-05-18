# Assinatura automática de módulos do VirtualBox com Secure Boot no Debian

Este repositório contém um script e um serviço `systemd` para assinar automaticamente os módulos do VirtualBox ao inicializar o sistema com o Secure Boot ativado.

## 🔐 Requisitos

- Debian ou derivado (Ubuntu, Zorin, Mint etc.)
- VirtualBox instalado (use o `.deb` oficial do site)
- Secure Boot ativado no UEFI/BIOS
- `mokutil`, `openssl`, `kmod`, `systemd`, `notify-send` instalados

---

## ⚙️ Passo a passo

### 1. Gere o par de chaves (MOK)

Abra o terminal e execute:

```bash
sudo openssl req -new -x509 -newkey rsa:2048 -keyout MOK.priv \
-outform DER -out MOK.der -nodes -days 36500 \
-subj "/CN=VirtualBox/"

```
🔐 Este comando gera as chaves para assinatura dos módulos:

- MOK.priv: chave privada

- MOK.der: certificado (em formato DER, usado pelo MOK Manager)

### 2. Registrar/Importar a chave para o Secure Boot (MOK)

```bash
sudo mokutil --import /var/lib/shim-signed/mok/MOK.der
```
Digite a senha para ser usada no próximo boot.

Esse comando agenda o registro da chave no Secure Boot, que será concluído na próxima reinicialização, quando você verá a tela azul do MOK Manager para confirmar a importação.

Você criará uma senha temporária. Na próxima reinicialização, use essa senha para concluir o registro da chave na tela azul do MOK Manager.

---


### 3. Reinicie o sistema e faça o “Enroll MOK” na tela azul durante o boot:

Escolha “Enroll MOK” → “Continue” → digite a senha → “Yes” → “Reboot”.

### 4. Assine os módulos novamente (agora com a MOK válida):

```sh
KDIR=/usr/src/linux-headers-$(uname -r)
PRIV=/var/lib/shim-signed/mok/MOK.priv
CERT=/var/lib/shim-signed/mok/MOK.der

for mod in vboxdrv vboxnetflt vboxnetadp; do
  FILE=$(modinfo -n $mod 2>/dev/null)
  if [ -f "$FILE" ]; then
    sudo $KDIR/scripts/sign-file sha256 $PRIV $CERT $FILE
  fi
done
```


### 5. Crie o script de assinatura automática

```bash
sudo nano /usr/local/bin/assinador_virtualbox.sh
```

Cole o seguinte conteúdo:

```bash

#!/bin/bash

MOK_DIR="/var/lib/shim-signed/mok"
PRIV="$MOK_DIR/MOK.priv"
CERT="$MOK_DIR/MOK.der"
KDIR="/usr/src/linux-headers-$(uname -r)"
SIGN="$KDIR/scripts/sign-file"

for mod in vboxdrv vboxnetflt vboxnetadp; do
  FILE=$(modinfo -n $mod 2>/dev/null)
  if [ -f "$FILE" ]; then
    SIGNATURE=$(modinfo "$FILE" | grep signer | cut -d ':' -f2 | xargs)
    if [[ "$SIGNATURE" != "VirtualBox Secure Boot" ]]; then
      echo "Assinando $mod..."
      "$SIGN" sha256 "$PRIV" "$CERT" "$FILE"
    fi
  fi
done
```

Depois, torne-o executável:

```bash
sudo chmod +x /usr/local/bin/assinador_virtualbox.sh
```

### 6. Crie o serviço systemd

```bash
sudo nano /etc/systemd/system/assinador-virtualbox.service
```

Cole o conteúdo abaixo:

```ini
[Unit]
Description=Assinatura automática dos módulos do VirtualBox
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/assinador_virtualbox.sh

[Install]
WantedBy=multi-user.target
```

Ative o serviço:

```bash
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl enable assinador-virtualbox.service
```

### 7. Mova as chaves para `/root`

```bash
sudo mv ~/MOK.* /root/
sudo chown root:root /root/MOK.*
chmod 600 /root/MOK.priv
```

---

## ✅ Verificando se o serviço está ativo

```bash
systemctl status assinador-virtualbox.service
```

Você pode ver os logs com:

```bash
journalctl -u assinador-virtualbox.service
```

---

## 🔁 Agora é só reiniciar

Sempre que você reiniciar, os módulos do VirtualBox serão assinados automaticamente (caso necessário) e você receberá uma notificação.

---

## 👤 Autor

Adaptado por [andrefonso](https://github.com/andrefonso)
