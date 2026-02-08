---
title: Синхронизация
description: 
published: true
date: 2026-02-08T17:32:39.035Z
tags: 
editor: markdown
dateCreated: 2026-02-08T17:32:39.035Z
---

Вот полный Python-скрипт для синхронизации через USB с подробными комментариями:

```python
#!/usr/bin/env python3
"""
USB-Syncthing Bridge Script

Скрипт создает мост между MTP-устройством (телефоном) и папкой Syncthing.
При подключении телефона по USB:
1. Монтирует телефон через jmtpfs
2. Синхронизирует файлы между телефоном и локальной папкой
3. Запускает мониторинг изменений в обе стороны
4. При отключении - размонтирует и ждет следующего подключения

Требования:
- python3
- jmtpfs
- rsync
- inotify-tools (inotifywait)
- syncthing (опционально)
"""

import os
import sys
import time
import signal
import subprocess
import threading
import logging
from pathlib import Path
import configparser
import json
from datetime import datetime
import re

class USBSyncthingBridge:
    def __init__(self, config_path="~/.config/usb-syncthing.conf"):
        """
        Инициализация моста синхронизации
        
        Args:
            config_path (str): Путь к файлу конфигурации
        """
        # Разворачиваем путь к домашней директории
        self.config_path = os.path.expanduser(config_path)
        
        # Инициализируем логирование
        self.setup_logging()
        
        # Флаг для контроля работы основного цикла
        self.running = True
        
        # Загружаем конфигурацию
        self.load_config()
        
        # Устанавливаем обработчик сигналов для graceful shutdown
        self.setup_signal_handlers()
        
        # Проверяем зависимости
        self.check_dependencies()
        
        # Словарь для хранения PID'ов запущенных процессов
        self.processes = {}
        
        logging.info("USB-Syncthing Bridge инициализирован")
        logging.info(f"Конфиг: {self.config_path}")
        logging.info(f"Устройство: {self.device_id}")
        logging.info(f"Точка монтирования: {self.mount_point}")
        logging.info(f"Папка на телефоне: {self.phone_dir}")
        logging.info(f"Локальная папка: {self.local_sync_dir}")
    
    def setup_logging(self):
        """Настройка системы логирования"""
        log_dir = os.path.expanduser("~/.local/share/usb-syncthing")
        os.makedirs(log_dir, exist_ok=True)
        
        log_file = os.path.join(log_dir, "bridge.log")
        
        logging.basicConfig(
            level=logging.INFO,
            format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
            handlers=[
                logging.FileHandler(log_file),
                logging.StreamHandler(sys.stdout)
            ]
        )
        
        self.logger = logging.getLogger(__name__)
    
    def load_config(self):
        """Загрузка конфигурации из файла"""
        # Значения по умолчанию
        default_config = {
            'device': {
                'vendor_id': '2717',
                'product_id': 'ff48',
                'name': 'Xiaomi Mi-2s'
            },
            'paths': {
                'mount_point': '~/media/android',
                'phone_dir': '/sdcard/Sync',
                'local_sync_dir': '~/Sync/Android'
            },
            'sync': {
                'exclude_patterns': '.*, *.tmp, *.temp, Thumbs.db',
                'sync_interval': '10',
                'two_way_sync': 'true'
            },
            'syncthing': {
                'auto_start': 'false',
                'syncthing_cmd': 'syncthing',
                'syncthing_args': '-no-browser'
            }
        }
        
        # Создаем конфиг-парсер
        config = configparser.ConfigParser()
        
        # Если файл конфига существует, читаем его
        if os.path.exists(self.config_path):
            config.read(self.config_path)
        else:
            # Создаем конфиг с значениями по умолчанию
            for section, options in default_config.items():
                config[section] = options
            
            # Сохраняем конфиг по умолчанию
            os.makedirs(os.path.dirname(self.config_path), exist_ok=True)
            with open(self.config_path, 'w') as f:
                config.write(f)
            
            self.logger.info(f"Создан конфигурационный файл: {self.config_path}")
            self.logger.info("Пожалуйста, отредактируйте его под ваше устройство")
        
        # Извлекаем настройки
        self.device_id = f"{config.get('device', 'vendor_id')}:{config.get('device', 'product_id')}"
        self.device_name = config.get('device', 'name')
        
        # Разворачиваем пути (заменяем ~ на домашнюю директорию)
        self.mount_point = os.path.expanduser(config.get('paths', 'mount_point'))
        self.phone_dir = config.get('paths', 'phone_dir')
        self.local_sync_dir = os.path.expanduser(config.get('paths', 'local_sync_dir'))
        
        # Создаем локальную папку если её нет
        os.makedirs(self.local_sync_dir, exist_ok=True)
        
        # Настройки синхронизации
        self.exclude_patterns = config.get('sync', 'exclude_patterns').split(', ')
        self.sync_interval = int(config.get('sync', 'sync_interval'))
        self.two_way_sync = config.getboolean('sync', 'two_way_sync')
        
        # Настройки Syncthing
        self.auto_start_syncthing = config.getboolean('syncthing', 'auto_start')
        self.syncthing_cmd = config.get('syncthing', 'syncthing_cmd')
        self.syncthing_args = config.get('syncthing', 'syncthing_args')
        
        # Дополнительные вычисляемые параметры
        self.full_phone_path = os.path.join(self.mount_point, self.phone_dir.lstrip('/'))
    
    def check_dependencies(self):
        """Проверка наличия необходимых программ"""
        dependencies = [
            ('jmtpfs', ['jmtpfs', '--version']),
            ('rsync', ['rsync', '--version']),
            ('inotifywait', ['which', 'inotifywait']),
        ]
        
        if self.auto_start_syncthing:
            dependencies.append(('syncthing', [self.syncthing_cmd, '--version']))
        
        missing_deps = []
        
        for dep_name, cmd in dependencies:
            try:
                result = subprocess.run(
                    cmd,
                    stdout=subprocess.PIPE,
                    stderr=subprocess.PIPE,
                    timeout=5
                )
                if result.returncode != 0:
                    missing_deps.append(dep_name)
            except (subprocess.TimeoutExpired, FileNotFoundError):
                missing_deps.append(dep_name)
        
        if missing_deps:
            self.logger.error(f"Отсутствуют зависимости: {', '.join(missing_deps)}")
            self.logger.info("Установите их командами:")
            self.logger.info("  sudo dnf install jmtpfs rsync inotify-tools")
            if 'syncthing' in missing_deps:
                self.logger.info("  sudo dnf install syncthing")
            sys.exit(1)
    
    def setup_signal_handlers(self):
        """Установка обработчиков сигналов для корректного завершения"""
        def signal_handler(signum, frame):
            self.logger.info(f"Получен сигнал {signum}, завершаем работу...")
            self.running = False
            self.cleanup()
            sys.exit(0)
        
        signal.signal(signal.SIGINT, signal_handler)
        signal.signal(signal.SIGTERM, signal_handler)
    
    def is_device_connected(self):
        """
        Проверка подключено ли устройство через USB
        
        Returns:
            bool: True если устройство подключено
        """
        try:
            result = subprocess.run(
                ['lsusb'],
                stdout=subprocess.PIPE,
                stderr=subprocess.PIPE,
                text=True,
                timeout=5
            )
            
            # Ищем наше устройство по VID:PID
            if self.device_id in result.stdout:
                self.logger.debug(f"Устройство {self.device_id} обнаружено")
                return True
            
            # Альтернативный поиск по имени
            if self.device_name and self.device_name in result.stdout:
                self.logger.debug(f"Устройство {self.device_name} обнаружено")
                return True
            
        except subprocess.TimeoutExpired:
            self.logger.error("Таймаут при выполнении lsusb")
        
        return False
    
    def mount_device(self):
        """
        Монтирование устройства через jmtpfs
        
        Returns:
            bool: True если монтирование успешно
        """
        # Проверяем, не смонтировано ли уже
        if self.is_mounted():
            self.logger.info(f"Устройство уже смонтировано в {self.mount_point}")
            return True
        
        # Убеждаемся, что точка монтирования существует
        os.makedirs(self.mount_point, exist_ok=True)
        
        # Останавливаем процессы, которые могут мешать (KDE/GVFS)
        self.stop_interfering_processes()
        
        # Монтируем устройство
        self.logger.info(f"Монтируем устройство в {self.mount_point}")
        
        try:
            result = subprocess.run(
                ['jmtpfs', self.mount_point],
                stdout=subprocess.PIPE,
                stderr=subprocess.PIPE,
                text=True,
                timeout=30
            )
            
            if result.returncode == 0:
                self.logger.info("Устройство успешно смонтировано")
                
                # Даем системе время на стабилизацию
                time.sleep(3)
                
                # Проверяем, что папка на телефоне существует
                phone_path = Path(self.full_phone_path)
                if not phone_path.exists():
                    self.logger.warning(f"Папка {self.phone_dir} не найдена на устройстве")
                    self.logger.info(f"Создаю папку {self.phone_dir}")
                    phone_path.mkdir(parents=True, exist_ok=True)
                
                return True
            else:
                self.logger.error(f"Ошибка монтирования: {result.stderr}")
                return False
                
        except subprocess.TimeoutExpired:
            self.logger.error("Таймаут при монтировании устройства")
            return False
    
    def unmount_device(self):
        """Размонтирование устройства"""
        if not self.is_mounted():
            self.logger.info("Устройство не монтировано")
            return True
        
        self.logger.info(f"Размонтируем устройство из {self.mount_point}")
        
        try:
            # Пытаемся размонтировать через fusermount
            result = subprocess.run(
                ['fusermount', '-u', self.mount_point],
                stdout=subprocess.PIPE,
                stderr=subprocess.PIPE,
                text=True,
                timeout=10
            )
            
            if result.returncode == 0:
                self.logger.info("Устройство успешно размонтировано")
                return True
            else:
                self.logger.warning(f"Не удалось размонтировать через fusermount: {result.stderr}")
                
                # Пробуем через umount
                result = subprocess.run(
                    ['umount', self.mount_point],
                    stdout=subprocess.PIPE,
                    stderr=subprocess.PIPE,
                    text=True
                )
                
                if result.returncode == 0:
                    self.logger.info("Устройство размонтировано через umount")
                    return True
                else:
                    self.logger.error(f"Не удалось размонтировать устройство: {result.stderr}")
                    return False
                    
        except Exception as e:
            self.logger.error(f"Исключение при размонтировании: {e}")
            return False
    
    def is_mounted(self):
        """
        Проверка смонтировано ли устройство
        
        Returns:
            bool: True если устройство смонтировано
        """
        try:
            # Проверяем через mountpoint
            result = subprocess.run(
                ['mountpoint', '-q', self.mount_point],
                stdout=subprocess.PIPE,
                stderr=subprocess.PIPE
            )
            return result.returncode == 0
        except:
            # Альтернативная проверка
            return os.path.ismount(self.mount_point)
    
    def stop_interfering_processes(self):
        """Остановка процессов, которые могут мешать монтированию (KDE/GVFS)"""
        processes_to_stop = ['kiod5', 'gvfsd-mtp', 'gvfs-gphoto2-volume-monitor']
        
        for proc_name in processes_to_stop:
            try:
                subprocess.run(
                    ['pkill', '-f', proc_name],
                    stdout=subprocess.DEVNULL,
                    stderr=subprocess.DEVNULL
                )
                self.logger.debug(f"Попытка остановить процесс: {proc_name}")
                time.sleep(1)
            except:
                pass
    
    def sync_phone_to_local(self):
        """
        Синхронизация с телефона на локальный компьютер
        
        Returns:
            bool: True если синхронизация успешна
        """
        self.logger.info(f"Синхронизация {self.phone_dir} -> {self.local_sync_dir}")
        
        # Строим команду rsync
        cmd = ['rsync', '-av', '--delete']
        
        # Добавляем исключения
        for pattern in self.exclude_patterns:
            if pattern.strip():
                cmd.extend(['--exclude', pattern.strip()])
        
        # Добавляем источник и назначение
        src = f"{self.full_phone_path}/"
        dst = f"{self.local_sync_dir}/"
        
        cmd.extend([src, dst])
        
        # Выполняем синхронизацию
        try:
            result = subprocess.run(
                cmd,
                stdout=subprocess.PIPE,
                stderr=subprocess.PIPE,
                text=True,
                timeout=300  # 5 минут на синхронизацию
            )
            
            if result.returncode == 0:
                self.logger.info("Синхронизация с телефона завершена успешно")
                if result.stdout:
                    self.logger.debug(f"Вывод rsync: {result.stdout[:500]}...")
                return True
            else:
                self.logger.error(f"Ошибка синхронизации: {result.stderr}")
                return False
                
        except subprocess.TimeoutExpired:
            self.logger.error("Таймаут при синхронизации с телефона")
            return False
    
    def sync_local_to_phone(self):
        """
        Синхронизация с локального компьютера на телефон
        
        Returns:
            bool: True если синхронизация успешна
        """
        self.logger.info(f"Синхронизация {self.local_sync_dir} -> {self.phone_dir}")
        
        # Строим команду rsync
        cmd = ['rsync', '-av', '--delete']
        
        # Добавляем исключения
        for pattern in self.exclude_patterns:
            if pattern.strip():
                cmd.extend(['--exclude', pattern.strip()])
        
        # Добавляем источник и назначение
        src = f"{self.local_sync_dir}/"
        dst = f"{self.full_phone_path}/"
        
        cmd.extend([src, dst])
        
        # Выполняем синхронизацию
        try:
            result = subprocess.run(
                cmd,
                stdout=subprocess.PIPE,
                stderr=subprocess.PIPE,
                text=True,
                timeout=300  # 5 минут на синхронизацию
            )
            
            if result.returncode == 0:
                self.logger.info("Синхронизация на телефон завершена успешно")
                return True
            else:
                self.logger.error(f"Ошибка синхронизации: {result.stderr}")
                return False
                
        except subprocess.TimeoutExpired:
            self.logger.error("Таймаут при синхронизации на телефон")
            return False
    
    def start_syncthing(self):
        """Запуск Syncthing если он не запущен"""
        if not self.auto_start_syncthing:
            return
        
        # Проверяем, запущен ли уже syncthing
        try:
            result = subprocess.run(
                ['pgrep', '-x', 'syncthing'],
                stdout=subprocess.PIPE,
                stderr=subprocess.PIPE
            )
            
            if result.returncode == 0:
                self.logger.info("Syncthing уже запущен")
                return
        except:
            pass
        
        # Запускаем syncthing
        self.logger.info("Запускаем Syncthing")
        
        cmd = [self.syncthing_cmd]
        if self.syncthing_args:
            cmd.extend(self.syncthing_args.split())
        
        try:
            # Запускаем в отдельном процессе
            process = subprocess.Popen(
                cmd,
                stdout=subprocess.DEVNULL,
                stderr=subprocess.DEVNULL,
                start_new_session=True
            )
            
            self.processes['syncthing'] = process.pid
            self.logger.info(f"Syncthing запущен с PID: {process.pid}")
            
        except Exception as e:
            self.logger.error(f"Не удалось запустить Syncthing: {e}")
    
    def start_file_monitoring(self):
        """
        Запуск мониторинга изменений файлов через inotifywait
        
        Запускается в отдельном потоке для каждой директории
        """
        if not self.two_way_sync:
            self.logger.info("Двусторонняя синхронизация отключена")
            return
        
        self.monitoring_running = True
        
        # Функция для мониторинга изменений
        def monitor_changes(source_dir, target_dir, sync_func, name):
            """Мониторинг изменений в директории"""
            self.logger.info(f"Запуск мониторинга {name}")
            
            # Команда inotifywait
            cmd = [
                'inotifywait',
                '-m',
                '-r',
                '-e', 'modify,create,delete,move',
                source_dir
            ]
            
            try:
                process = subprocess.Popen(
                    cmd,
                    stdout=subprocess.PIPE,
                    stderr=subprocess.PIPE,
                    text=True,
                    bufsize=1,
                    universal_newlines=True
                )
                
                self.processes[f'inotify_{name}'] = process.pid
                
                # Читаем вывод построчно
                for line in process.stdout:
                    if not self.monitoring_running:
                        break
                    
                    # Парсим событие
                    if line.strip():
                        self.logger.debug(f"Событие {name}: {line.strip()}")
                        
                        # Ждем немного, чтобы накопились изменения
                        time.sleep(2)
                        
                        # Запускаем синхронизацию
                        sync_func()
                
                process.terminate()
                
            except Exception as e:
                self.logger.error(f"Ошибка мониторинга {name}: {e}")
        
        # Запускаем мониторинг в отдельных потоках
        threads = []
        
        # Мониторинг телефона → компьютер
        if Path(self.full_phone_path).exists():
            t1 = threading.Thread(
                target=monitor_changes,
                args=(
                    self.full_phone_path,
                    self.local_sync_dir,
                    self.sync_phone_to_local,
                    'phone_to_local'
                ),
                daemon=True
            )
            threads.append(t1)
        
        # Мониторинг компьютер → телефон
        t2 = threading.Thread(
            target=monitor_changes,
            args=(
                self.local_sync_dir,
                self.full_phone_path,
                self.sync_local_to_phone,
                'local_to_phone'
            ),
            daemon=True
        )
        threads.append(t2)
        
        # Запускаем все потоки
        for t in threads:
            t.start()
        
        self.logger.info("Мониторинг файлов запущен")
        
        return threads
    
    def initial_sync(self):
        """Выполнение первоначальной синхронизации"""
        self.logger.info("Начинаем первоначальную синхронизацию")
        
        # Создаем файл статуса синхронизации
        status_file = Path(self.local_sync_dir) / '.sync_status.json'
        status = {
            'last_sync_from_phone': None,
            'last_sync_to_phone': None,
            'last_full_sync': None
        }
        
        if status_file.exists():
            try:
                with open(status_file, 'r') as f:
                    status = json.load(f)
            except:
                pass
        
        # Проверяем, какая сторона новее
        phone_newer = False
        local_newer = False
        
        # Простая эвристика - сравниваем время модификации
        try:
            phone_mtime = os.path.getmtime(self.full_phone_path)
            local_mtime = os.path.getmtime(self.local_sync_dir)
            
            if phone_mtime > local_mtime:
                phone_newer = True
            elif local_mtime > phone_mtime:
                local_newer = True
        except:
            # Если не удалось определить, синхронизируем в обе стороны
            phone_newer = True
            local_newer = True
        
        # Выполняем синхронизацию
        sync_success = True
        
        if phone_newer:
            if self.sync_phone_to_local():
                status['last_sync_from_phone'] = datetime.now().isoformat()
            else:
                sync_success = False
        
        if local_newer and self.two_way_sync:
            if self.sync_local_to_phone():
                status['last_sync_to_phone'] = datetime.now().isoformat()
            else:
                sync_success = False
        
        if sync_success:
            status['last_full_sync'] = datetime.now().isoformat()
            
            # Сохраняем статус
            with open(status_file, 'w') as f:
                json.dump(status, f, indent=2)
            
            self.logger.info("Первоначальная синхронизация завершена")
        
        return sync_success
    
    def cleanup(self):
        """Очистка ресурсов и завершение работы"""
        self.logger.info("Очистка ресурсов...")
        
        # Останавливаем мониторинг
        self.monitoring_running = False
        
        # Завершаем все запущенные процессы
        for name, pid in list(self.processes.items()):
            try:
                self.logger.info(f"Останавливаем процесс {name} (PID: {pid})")
                subprocess.run(['pkill', '-P', str(pid)], 
                             stdout=subprocess.DEVNULL, 
                             stderr=subprocess.DEVNULL)
            except:
                pass
        
        # Размонтируем устройство
        self.unmount_device()
        
        self.logger.info("Очистка завершена")
    
    def run(self):
        """Основной цикл работы программы"""
        self.logger.info("Запуск USB-Syncthing Bridge")
        
        last_device_state = False
        
        while self.running:
            try:
                # Проверяем состояние устройства
                device_connected = self.is_device_connected()
                
                # Если состояние изменилось
                if device_connected != last_device_state:
                    if device_connected:
                        self.logger.info(f"Устройство {self.device_id} подключено")
                        
                        # Монтируем устройство
                        if self.mount_device():
                            # Первоначальная синхронизация
                            if self.initial_sync():
                                # Запускаем Syncthing
                                self.start_syncthing()
                                
                                # Запускаем мониторинг файлов
                                self.monitoring_threads = self.start_file_monitoring()
                            else:
                                self.logger.error("Не удалось выполнить первоначальную синхронизацию")
                        else:
                            self.logger.error("Не удалось смонтировать устройство")
                    else:
                        self.logger.info("Устройство отключено")
                        
                        # Останавливаем мониторинг
                        self.monitoring_running = False
                        
                        # Размонтируем устройство
                        self.unmount_device()
                
                # Обновляем состояние
                last_device_state = device_connected
                
                # Ждем перед следующей проверкой
                time.sleep(self.sync_interval)
                
            except KeyboardInterrupt:
                self.logger.info("Получен сигнал прерывания")
                break
            except Exception as e:
                self.logger.error(f"Ошибка в основном цикле: {e}", exc_info=True)
                time.sleep(10)
        
        # Завершаем работу
        self.cleanup()
        self.logger.info("USB-Syncthing Bridge остановлен")

def main():
    """Точка входа в программу"""
    # Проверяем аргументы командной строки
    config_path = "~/.config/usb-syncthing.conf"
    
    if len(sys.argv) > 1:
        if sys.argv[1] in ['-h', '--help']:
            print("USB-Syncthing Bridge - синхронизация файлов через USB")
            print("\nИспользование:")
            print("  python3 usb_syncthing_bridge.py [КОНФИГ_ФАЙЛ]")
            print("\nАргументы:")
            print("  КОНФИГ_ФАЙЛ - путь к файлу конфигурации (по умолчанию: ~/.config/usb-syncthing.conf)")
            print("\nПример:")
            print("  python3 usb_syncthing_bridge.py")
            print("  python3 usb_syncthing_bridge.py ~/my_config.conf")
            sys.exit(0)
        else:
            config_path = sys.argv[1]
    
    # Создаем и запускаем мост
    bridge = USBSyncthingBridge(config_path)
    bridge.run()

if __name__ == "__main__":
    main()
```

## Дополнительные файлы:

### 1. **Конфигурационный файл (`~/.config/usb-syncthing.conf`)**:
```ini
[device]
# VID и PID устройства (из lsusb)
vendor_id = 2717
product_id = ff48
name = Xiaomi Mi-2s

[paths]
# Точка монтирования телефона
mount_point = ~/media/android

# Папка на телефоне для синхронизации
phone_dir = /sdcard/Sync

# Локальная папка для Syncthing
local_sync_dir = ~/Sync/Android

[sync]
# Паттерны файлов для исключения
exclude_patterns = .*, *.tmp, *.temp, Thumbs.db, .DS_Store, *.log

# Интервал проверки подключения (секунды)
sync_interval = 10

# Двусторонняя синхронизация
two_way_sync = true

[syncthing]
# Автоматически запускать Syncthing
auto_start = false

# Команда запуска Syncthing
syncthing_cmd = syncthing

# Аргументы командной строки
syncthing_args = -no-browser
```

### 2. **Скрипт установки как systemd службы (`install-service.sh`)**:
```bash
#!/bin/bash
# Установка USB-Syncthing Bridge как systemd службы

SERVICE_NAME="usb-syncthing-bridge"
SCRIPT_PATH="/usr/local/bin/usb_syncthing_bridge.py"
SERVICE_FILE="/etc/systemd/system/$SERVICE_NAME.service"
USER=$(whoami)

# Копируем скрипт
sudo cp usb_syncthing_bridge.py $SCRIPT_PATH
sudo chmod +x $SCRIPT_PATH

# Создаем service файл
echo "[Unit]
Description=USB-Syncthing Bridge Service
After=network.target

[Service]
Type=simple
User=$USER
ExecStart=/usr/bin/python3 $SCRIPT_PATH
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target" | sudo tee $SERVICE_FILE

# Перезагружаем systemd и запускаем службу
sudo systemctl daemon-reload
sudo systemctl enable $SERVICE_NAME
sudo systemctl start $SERVICE_NAME

echo "Служба установлена и запущена"
echo "Проверить статус: sudo systemctl status $SERVICE_NAME"
echo "Логи: journalctl -u $SERVICE_NAME -f"
```

### 3. **Скрипт для ручного запуска (`start-bridge.sh`)**:
```bash
#!/bin/bash
# Ручной запуск USB-Syncthing Bridge

# Переходим в директорию скрипта
cd "$(dirname "$0")"

# Проверяем зависимости
if ! command -v python3 &> /dev/null; then
    echo "Ошибка: python3 не установлен"
    exit 1
fi

# Запускаем скрипт
python3 usb_syncthing_bridge.py "$@"
```

## Особенности скрипта:

1. **Асинхронный мониторинг** - использует потоки для одновременного отслеживания изменений
2. **Graceful shutdown** - корректно обрабатывает прерывания и сигналы
3. **Логирование** - детальные логи в файл и консоль
4. **Конфигурируемость** - все настройки в отдельном файле
5. **Проверка зависимостей** - автоматическая проверка наличия необходимых программ
6. **Обработка ошибок** - устойчивость к сбоям монтирования/синхронизации
7. **Статус синхронизации** - сохранение информации о последних синхронизациях

## Установка и использование:

```bash
# 1. Сделайте скрипт исполняемым
chmod +x usb_syncthing_bridge.py

# 2. Отредактируйте конфиг под ваше устройство
nano ~/.config/usb-syncthing.conf

# 3. Запустите скрипт
./usb_syncthing_bridge.py

# 4. Или установите как службу
sudo ./install-service.sh
```

Скрипт будет автоматически обнаруживать подключение телефона, монтировать его и синхронизировать файлы с локальной папкой, которую можно использовать с Syncthing.