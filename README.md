# firmware_bin


Binários para atualização do firmware Beo v2.x


```sh
#!/bin/bash

set -e

# Diretório raiz do repositório (hardware)
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
BASE_DIR="$SCRIPT_DIR"

# Diretórios de build e upload
BUILD_DIR="$BASE_DIR/Firmware/build"
REPO_DIR="$BASE_DIR/firmware_bin"
UPLOAD_DIR="$REPO_DIR/bins"


# 2. Verifica os direitos de acesso ao repositório dos binários 
REPO_URL="git@github.com:Qiron-Robotics/firmware_bin.git"


# Flag de controle para execução do processo de flash
FLASH=false

# Porta USB para upload via ESPTOOL 
USB_PORT=""


# Verifica os argumentos de linha de comando 
while [[ $# -gt 0 ]]; do
    case "$1" in
        --flash)
            FLASH=true
            shift
            ;;
        --port)
            if [ -n "$2" ]; then
                USB_PORT="$2"
                echo "Porta USB definida manualmente: $USB_PORT"
                shift 2
            else
                echo "Erro: --port requer um argumento."
                exit 1
            fi
            ;;
        *)
            echo "Argumento desconhecido: $1"
            echo "Uso: $0 [--flash] [--port <porta_usb>]"
            exit 1
            ;;
    esac
done


# Verifica a USB_PORT
if [ -z "$USB_PORT" ]; then
    USB_PORT=$(ls -t /dev/ttyUSB* /dev/ttyACM* 2>/dev/null | head -n 1)
    if [ -n "$USB_PORT" ]; then
        echo "Usando porta detectada: $USB_PORT"
    else
        echo "Erro: Nenhuma porta USB detectada. Use --port para especificar manualmente."
        exit 1
    fi
fi


# Verifica o acesso ao repositório de binários do firmware
if ! git ls-remote "$REPO_URL" > /dev/null 2>&1; then
    echo "Erro: Sem acesso ao repositório."
    exit 1
fi


if [ ! -d "$REPO_DIR/.git" ]; then
    echo "Clonando repositório..."
    rm -rf "$REPO_DIR"
    git clone "$REPO_URL" "$REPO_DIR"
fi


# 1. Atualiza os binários do firmware para o repositório
if [ ! -d "$UPLOAD_DIR" ]; then
    mkdir -p "$UPLOAD_DIR"
fi

FILES=(
    "$BUILD_DIR/QiCore.bin"
    "$BUILD_DIR/partition_table/partition-table.bin"
    "$BUILD_DIR/bootloader/bootloader.bin"
)

for file in "${FILES[@]}"; do
    if [ -f "$file" ]; then
        cp -f "$file" "$UPLOAD_DIR/"
        echo "Copiado: $(basename "$file")"
    else
        echo "Aviso: arquivo não encontrado -> $file"
    fi
done

# Commit e push dos binários para o repositório
cd "$REPO_DIR"
git add bins/
git commit -m "Atualização automática dos binários do firmware" \
    || echo "Nada para commit"

git push origin main
cd "$BASE_DIR"


# Executa o script de upload para o microcontrolador

if [ "$FLASH" = true ]; then
    echo "Iniciando processo de flash..."

    PYTHON_CMD="python3"

    # Verifica se existe venv local
    if [ -f "$BASE_DIR/.venv/bin/python3" ]; then
        echo "Usando Python do venv local."
        PYTHON_CMD="$BASE_DIR/.venv/bin/python3"
    fi

    # Detecta se é Raspberry Pi
    if grep -q "Raspberry Pi" /proc/device-tree/model 2>/dev/null; then
        echo "Rodando em Raspberry Pi"

        DO_RESET_BEFORE="no-reset"
        DO_RESET_AFTER="no-reset"

        BOOT="gpiochip4 17"
        RESET="gpiochip4 27"

        echo "Colocando em modo boot..."
        gpioset $BOOT=0

        echo "Resetando..."
        gpioset $RESET=0
        sleep 0.5
        gpioset $RESET=1
        sleep 1.0

    else
        echo "Rodando em PC ou outro hardware"

        DO_RESET_BEFORE="default-reset"
        DO_RESET_AFTER="hard-reset"
    fi

    echo "Executando flash..."

    $PYTHON_CMD -m esptool \
        -p "$USB_PORT" \
        -b 460800 \
        --before "$DO_RESET_BEFORE" \
        --after "$DO_RESET_AFTER" \
        --chip esp32 \
        write-flash \
        --flash-mode dio \
        --flash-freq 40m \
        --flash-size detect \
        0x1000  "$UPLOAD_DIR/bootloader.bin" \
        0x20000 "$UPLOAD_DIR/QiCore.bin" \
        0x8000  "$UPLOAD_DIR/partition-table.bin"

    # Reset final se necessário
    if [ "$DO_RESET_AFTER" == "no-reset" ]; then
        echo "Reiniciando dispositivo..."
        gpioset $RESET=0
        sleep 0.1
        gpioset $RESET=1
    fi

else
    echo "Flash ignorado. Use --flash para executar."
fi

echo "Processo finalizado com sucesso."
```
