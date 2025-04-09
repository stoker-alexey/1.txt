#!/usr/bin/env python3
# -*- coding: utf-8 -*-

"""
Простой клиент для GigaChat API с использованием http.client.
"""

import http.client
import json
import os
import ssl

# --- КОНФИГУРАЦИЯ ---
# Вставьте ваш токен авторизации GigaChat здесь
# Он должен начинаться с "Bearer "
GIGACHAT_API_TOKEN = ""  # <--- ВСТАВЬТЕ ТОКЕН СЮДА

GIGACHAT_API_HOST = "gigachat.devices.sberbank.ru"
GIGACHAT_API_PATH = "/api/v1/chat/completions"
GIGACHAT_MODEL = "GigaChat"  # Вы можете выбрать другую модель, если нужно
# --- КОНЕЦ КОНФИГУРАЦИИ ---

# Функция для получения токена авторизации
def get_gigachat_token():
    """
    Получает токен GigaChat из переменной GIGACHAT_API_TOKEN.
    Проверяет наличие токена и его формат.
    
    Returns:
        str: Токен авторизации или None, если токен некорректен.
    """
    token = GIGACHAT_API_TOKEN

    if not token or token == "Bearer ВАШ_ТОКЕН_ЗДЕСЬ":
        print("Ошибка: Пожалуйста, вставьте ваш токен GigaChat в переменную GIGACHAT_API_TOKEN в коде.")
        return None
    
    # Убедимся, что токен начинается с "Bearer "
    if not token.startswith("Bearer "):
        print("Предупреждение: Токен должен начинаться с 'Bearer '. Пожалуйста, проверьте формат токена.")
        # Можно либо добавить префикс автоматически, либо вернуть ошибку.
        # Вернем ошибку, чтобы пользователь точно указал правильный токен.
        # token = f"Bearer {token}" 
        return None
        
    return token

# Функция для отправки запроса к GigaChat
def send_to_gigachat(token, messages):
    """
    Отправляет запрос к API GigaChat и возвращает ответ.
    
    Args:
        token (str): Токен авторизации.
        messages (list): Список сообщений для отправки.
    
    Returns:
        dict: Ответ от API GigaChat или None в случае ошибки.
    """
    payload = json.dumps({
        "model": GIGACHAT_MODEL,
        "messages": messages,
        "profanity_check": True  # Включим проверку на ненормативную лексику
    })
    
    headers = {
        'Content-Type': 'application/json',
        'Accept': 'application/json',
        'Authorization': token
    }
    
    try:
        # Создаем безопасное соединение (важно для HTTPS)
        # Отключаем проверку сертификата для тестовых целей (не рекомендуется для продакшена!)
        context = ssl._create_unverified_context()
        conn = http.client.HTTPSConnection(GIGACHAT_API_HOST, context=context)
        
        conn.request("POST", GIGACHAT_API_PATH, payload, headers)
        res = conn.getresponse()
        
        if res.status != 200:
            print(f"Ошибка API GigaChat: Статус {res.status}, Причина: {res.reason}")
            error_data = res.read().decode("utf-8")
            print(f"Ответ ошибки: {error_data}")
            return None
            
        data = res.read()
        conn.close()  # Важно закрыть соединение
        
        # Декодируем и разбираем JSON ответ
        response_json = json.loads(data.decode("utf-8"))
        return response_json
        
    except http.client.HTTPException as e:
        print(f"Ошибка HTTP соединения: {e}")
        return None
    except json.JSONDecodeError as e:
        print(f"Ошибка декодирования JSON ответа: {e}")
        print(f"Полученные данные: {data.decode('utf-8') if 'data' in locals() else 'Нет данных'}")
        return None
    except Exception as e:
        print(f"Неожиданная ошибка: {e}")
        return None

# Основная функция
def main():
    """
    Основная функция программы: запрашивает токен и общается с GigaChat.
    """
    print("Запуск простого клиента GigaChat...")
    
    token = get_gigachat_token()
    if not token:
        print("Не удалось получить токен GigaChat. Выход.")
        return
        
    print("Клиент готов к работе! Введите 'выход' для завершения.")
    
    # Инициализируем историю сообщений
    message_history = []
    
    while True:
        user_input = input("\nВаш вопрос: ")
        
        if user_input.lower() in ["выход", "exit", "quit", "q"]:
            print("До свидания!")
            break
            
        # Добавляем сообщение пользователя в историю
        message_history.append({"role": "user", "content": user_input})
        
        # Отправляем всю историю сообщений (включая текущее)
        response = send_to_gigachat(token, message_history)
        
        if response and response.get("choices"):
            try:
                assistant_message = response["choices"][0]["message"]
                print(f"\nОтвет GigaChat:\n{assistant_message.get('content', 'Нет содержимого')}")
                
                # Добавляем ответ ассистента в историю
                message_history.append(assistant_message)
                
            except (IndexError, KeyError) as e:
                print(f"Не удалось извлечь ответ из JSON: {e}")
                print(f"Полный ответ: {response}")
        else:
            print("Не удалось получить корректный ответ от GigaChat.")
            # В случае ошибки, удаляем последнее сообщение пользователя из истории,
            # чтобы не отправлять его повторно при следующей итерации.
            if message_history:
                message_history.pop()

# Точка входа в программу
if __name__ == "__main__":
    main() 
