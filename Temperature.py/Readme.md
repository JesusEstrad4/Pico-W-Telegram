```

#Este codigo esta basado en el codigo de Max Antony

#Luis Jesus Estrada Jimenez

#Desarrollado con Micropyton
import struct
import socket
import network
import utime
import time
import gc
import ujson
import urequests
import utime
from machine import ADC
wifi_config = {
    'ssid':'NOMBRE WIFI',
    'password':'CLAVE WIFI'
}


utelegram_config = {
    'token': 'TOKEN DEl BOT'
}


class ubot:
    
    def __init__(self, token, offset=0):
        self.url = 'https://api.telegram.org/bot' + token
        self.commands = {}
        self.default_handler = None
        self.message_offset = offset
        self.sleep_btw_updates = 3

        messages = self.read_messages()
        if messages:
            if self.message_offset==0:
                self.message_offset = messages[-1]['update_id']
            else:
                for message in messages:
                    if message['update_id'] >= self.message_offset:
                        self.message_offset = message['update_id']
                        break


    def send(self, chat_id, text):
        data = {'chat_id': chat_id, 'text': text}
        try:
            headers = {'Content-type': 'application/json', 'Accept': 'text/plain'}
            response = urequests.post(self.url + '/sendMessage', json=data, headers=headers)
            response.close()
            return True
        except:
            return False

    def read_messages(self):
        result = []
        self.query_updates = {
            'offset': self.message_offset + 1,
            'limit': 1,
            'timeout': 30,
            'allowed_updates': ['message']}

        try:
            update_messages = urequests.post(self.url + '/getUpdates', json=self.query_updates).json() 
            if 'result' in update_messages:
                for item in update_messages['result']:
                    result.append(item)
            return result
        except (ValueError):
            return None
        except (OSError):
            print("OSError: request timed out")
            return None

    def listen(self):
        while True:
            self.read_once()
            time.sleep(self.sleep_btw_updates)
            gc.collect()

    def read_once(self):
        messages = self.read_messages()
        if messages:
            if self.message_offset==0:
                self.message_offset = messages[-1]['update_id']
                self.message_handler(messages[-1])
            else:
                for message in messages:
                    if message['update_id'] >= self.message_offset:
                        self.message_offset = message['update_id']
                        self.message_handler(message)
                        break
    
    def register(self, command, handler):
        self.commands[command] = handler

    def set_default_handler(self, handler):
        self.default_handler = handler

    def set_sleep_btw_updates(self, sleep_time):
        self.sleep_btw_updates = sleep_time

    def message_handler(self, message):
        if 'text' in message['message']:
            parts = message['message']['text'].split(' ')
            if parts[0] in self.commands:
                self.commands[parts[0]](message)
            else:
                if self.default_handler:
                    self.default_handler(message)



debug = True

sta_if = network.WLAN(network.STA_IF)
sta_if.active(True)
sta_if.scan()
sta_if.connect(wifi_config['ssid'], wifi_config['password'])

if debug: print('WAITING FOR NETWORK - sleep 20')
print(sta_if.isconnected())
while(not sta_if.isconnected()):
    print("red no disponible aun")
    utime.sleep(1)

def get_message(message):
    bot.send(message['message']['chat']['id'], message['message']['text'].upper())

def temp(message):
    #bot.send(message['message']['chat']['id'], '')
    sensor= ADC(4)
    valor=sensor.read_u16()*3.3/65535
    temp= 27-(valor-0.706)/0.001721
    print(temp)
    utime.sleep(1)
    bot.send(message['message']['chat']['id'], temp)        
    gc.collect()
    
    if temp>24:
        bot.send(message['message']['chat']['id'], 'TEMPERATURA ALTA')
    else:
        bot.send(message['message']['chat']['id'], 'TEMPERATURA NORMAL')

if sta_if.isconnected():
    print('conectando a telegram')
    bot = ubot(utelegram_config['token'])
    print('conexion a bot exitosa')
    bot.register('/temp', temp)
    
    bot.set_default_handler(get_message)

    print('BOT LISTENING')
    bot.listen()
else:
    print('NOT CONNECTED - aborting')

```
