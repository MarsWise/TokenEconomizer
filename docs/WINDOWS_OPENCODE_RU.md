# Запуск TokenEconomizer с OpenCode на Windows (PowerShell)

> Инструкция для Windows 11, PowerShell и OpenCode 2.x. Это дополнение к [основному README](../README.md), а не самостоятельная замена оригинальной документации Microsoft.

TokenEconomizer — плагин для кодинг-агентов, который использует опыт предыдущих сессий в одном репозитории. Для OpenCode заявлены **micro-skills** (память о проверенных особенностях проекта) и **loop guard** (выявление зацикливания); **tool pruning в OpenCode пока не реализован**. Это не ускоритель инференса и не способ повысить tokens/sec локальной LLM.

**Статус совместимости:** интеграция с OpenCode экспериментальная. Разработчики проверяли версии **1.19.5 и 2.2.0**; OpenCode **2.2.5** работает у автора инструкции, но совместимость плагина с ней отдельно не подтверждена. Успешный запуск OpenCode сам по себе **не доказывает**, что хуки TokenEconomizer работают.

## 1. Требования

- Windows 10/11 и PowerShell.
- Git, Python **3.10+**, установленный OpenCode CLI.
- Для создания символической ссылки: включённый **Developer Mode** Windows или PowerShell от администратора.
- Настроенная модель и провайдер OpenCode (можно использовать локальную модель; её настройка не входит в эту инструкцию).

Проверить:

```powershell
git --version
python --version
opencode --version
where.exe opencode
```

## 2. Скачать репозиторий

Пример с форком **MarsWise** (если уже клонирован — этот шаг пропустить):

```powershell
New-Item -ItemType Directory -Force "$HOME\OpenCodeWorkspace" | Out-Null
Set-Location "$HOME\OpenCodeWorkspace"
git clone https://github.com/MarsWise/TokenEconomizer.git
Set-Location .\TokenEconomizer
```

Не удаляйте и не переносите каталог после установки: ссылка на плагин должна указывать на исходный файл.

## 3. Подключить плагин к OpenCode

**Выполнить в PowerShell**, находясь в корне клона `TokenEconomizer`:

```powershell
$source = (Resolve-Path ".\plugin\opencode\tokeneconomizer.ts").Path
$targetDir = Join-Path $HOME ".config\opencode\plugins"
$target = Join-Path $targetDir "tokeneconomizer.ts"

New-Item -ItemType Directory -Force -Path $targetDir | Out-Null

if (Test-Path -LiteralPath $target) {
    Write-Host "Плагин уже существует: $target. Проверьте ссылку перед заменой."
} else {
    New-Item -ItemType SymbolicLink -Path $target -Target $source
}

Get-Item -LiteralPath $target | Format-List FullName,LinkType,Target
Test-Path -LiteralPath $target
```

Ожидается `True`; `LinkType` должен быть `SymbolicLink`, а `Target` — путь к `plugin\opencode\tokeneconomizer.ts`.

**Не используйте** Linux-команду `ln -s ... \` в PowerShell: здесь `ln` может оказаться `ln.exe` из другой программы (например, Cadence), а `\` не переносит команду на следующую строку как в Bash.

При ошибке доступа к символическим ссылкам включите **Settings → System → For developers → Developer Mode** или откройте PowerShell от администратора.

## 4. Запустить OpenCode из своего рабочего проекта

Плагин расположен в отдельном каталоге; **работать нужно из репозитория, который вы хотите анализировать**:

```powershell
Set-Location "C:\path\to\your-project"
opencode --standalone
```

Если нужен облегчённый интерфейс:

```powershell
opencode mini --standalone
```

Проверка без интерактивного интерфейса (при настроенной модели):

```powershell
opencode run "Reply with exactly OK" --standalone
```

Завершите несколько обычных рабочих сессий: TokenEconomizer учится на выполненной работе, а не только на факте установки.

## 5. Проверить TokenEconomizer и включить micro-skills

**Откройте второй PowerShell в корне нужного рабочего проекта**, а не в исходниках TokenEconomizer. Укажите реальный путь к вашему клону:

```powershell
$te = "$HOME\OpenCodeWorkspace\TokenEconomizer\plugin\scripts\te_bootstrap.py"

python $te cli --host opencode doctor
python $te cli --host opencode status
python $te cli --host opencode config
```

Чтобы включить память для **текущего рабочего репозитория**:

```powershell
python $te cli --host opencode config set microskills=on
python $te cli --host opencode microskills
```

Чтобы включить её во **всех** репозиториях:

```powershell
python $te cli --host opencode config set microskills=on --global
```

Вариант без дополнительных вызовов модели для генерации фактов:

```powershell
python $te cli --host opencode config set microskills=envfacts
```

Micro-skills по умолчанию выключены; их подтверждение требует нескольких сессий. Loop guard по умолчанию включён и начинает эффективно работать после накопления истории успешных задач. Все команды `te` для OpenCode выполняются **из терминала**: slash-команд этого плагина в OpenCode нет.

## 6. Диагностика

Если интерфейс не появляется или процесс сразу завершается:

```powershell
opencode --version
opencode --standalone --print-logs --log-level debug
Write-Host "Exit code: $LASTEXITCODE"
```

Проверьте и журнал плагина:

```powershell
$log = Join-Path $HOME ".tokeneconomizer\errors.log"
if (Test-Path $log) { Get-Content $log -Tail 80 }
```

Если `doctor` показывает ошибку импорта, убедитесь, что существует каталог `tokeneconomizer` в клоне, что Python версии 3.10+ и что символическая ссылка не указывает на удалённый каталог.

**Особенность OpenCode 2.x:** плагины выполняются в фоновом сервисе. Изменения переменных окружения `TE_*`, установленных в терминале, могут потребовать перезапуска сервиса:

```powershell
opencode service restart
```

Настройки `te config` не требуют перезапуска сервиса.

### Как временно отключить плагин

Для проверки проблем загрузки можно убрать только пользовательскую ссылку, **не трогая исходники**:

```powershell
$plugin = Join-Path $HOME ".config\opencode\plugins\tokeneconomizer.ts"
Rename-Item -LiteralPath $plugin -NewName "tokeneconomizer.disabled"
opencode service restart
```

Чтобы включить обратно:

```powershell
Rename-Item -LiteralPath (Join-Path $HOME ".config\opencode\plugins\tokeneconomizer.disabled") -NewName "tokeneconomizer.ts"
opencode service restart
```

Не отключайте плагин в середине незавершённой задачи. После изменений начинайте новую сессию.

## 7. Обновление

Из клона:

```powershell
Set-Location "$HOME\OpenCodeWorkspace\TokenEconomizer"
git pull --ff-only
```

> Если вы вносили собственные изменения в этот форк, сначала сохраните их и проверьте состояние `git status`. Обновление форка и синхронизация с upstream Microsoft — отдельная операция.

## Ссылки

- [Оригинальный проект Microsoft](https://github.com/microsoft/TokenEconomizer)
- [Оригинальная инструкция для плагина OpenCode](https://github.com/microsoft/TokenEconomizer/blob/main/plugin/README.md#opencode)
- [Форк MarsWise/TokenEconomizer](https://github.com/MarsWise/TokenEconomizer)
