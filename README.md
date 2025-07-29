# uzakdan-kap-acma

from PyQt5.QtWidgets import (
    QApplication, QWidget, QPushButton, QVBoxLayout, QLabel, QFrame,
    QHBoxLayout
)
from PyQt5.QtCore import Qt, QTimer
from PyQt5.QtGui import QIcon
from pymodbus.client import ModbusTcpClient
from PyQt5.QtMultimedia import QSound
import subprocess
import sys

# PLC bilgileri
PLC_IP = '192.168.5.250'
PLC_PORT = 502
COIL_ADDRESS = 2048
ZIL_COIL_ADDRESS = 2098
LED1280_ADDRESS = 1280

def ping_device(ip):
    try:
        creationflags = 0
        if sys.platform.startswith("win"):
            creationflags = subprocess.CREATE_NO_WINDOW

        subprocess.check_output(
            ['ping', '-n', '1', '-w', '200', ip],
            stderr=subprocess.DEVNULL,
            creationflags=creationflags
        )
        return True
    except subprocess.CalledProcessError:
        return False

class ModbusApp(QWidget):
    def __init__(self):
        super().__init__()
        self.setWindowTitle("Bien KapÄ± KontrolÃ¼")
        self.setWindowIcon(QIcon("favicon.ico"))      # Ä°KON EKLENDÄ°
        self.setFixedSize(360, 300)

        self.client = ModbusTcpClient(PLC_IP, port=PLC_PORT)
        self.client.connect()

        self.sound_looping = False

        # Buton + LED1280 yanyana
        self.button = QPushButton("KapÄ±yÄ± AÃ§Ä±nÄ±z", self)
        self.button.setCheckable(True)
        self.button.setStyleSheet("font-size: 20px; padding: 15px;")
        self.button.setMinimumHeight(60)
        self.button.setFixedWidth(200)
        self.button.pressed.connect(self.turn_on_output)
        self.button.released.connect(self.turn_off_output)

        self.led1280 = QLabel(self)
        self.led1280.setFixedSize(30, 30)
        self.led1280.setStyleSheet(
            "background-color: red; "
            "border-radius: 15px; "
            "border: 2px solid black;"
        )

        button_layout = QHBoxLayout()
        button_layout.addWidget(self.button)
        button_layout.addWidget(self.led1280)
        button_layout.addStretch()

        # PLC durumu
        self.status_label = QLabel("", self)
        self.status_label.setAlignment(Qt.AlignCenter)
        self.status_label.setStyleSheet("font-size: 14px; font-weight: bold;")

        # Zil LED
        self.led_label = QLabel("Zil Durumu:", self)
        self.led_indicator = QLabel(self)
        self.led_indicator.setFixedSize(20, 20)
        self.led_indicator.setStyleSheet(
            "background-color: red; "
            "border-radius: 10px; "
            "border: 1px solid black;"
        )

        led_layout = QHBoxLayout()
        led_layout.addWidget(self.led_label)
        led_layout.addWidget(self.led_indicator)
        led_layout.addStretch()

        # Bilgi bÃ¶lÃ¼mÃ¼ (Ã§erÃ§evesiz)
        self.info_frame = QFrame(self)
        self.info_frame.setFrameShape(QFrame.NoFrame)
        self.info_frame.setStyleSheet("background-color: transparent;")

        self.info_label = QLabel(
            f" PLC IP : {PLC_IP}\n  Bien Elektronik BAKIM ",
            self.info_frame
        )
        self.info_label.setAlignment(Qt.AlignCenter)
        self.info_label.setStyleSheet(
            "font-size: 14px; padding: 4px; "
            "font-weight: bold; font-family: Arial;"
        )

        info_layout = QVBoxLayout()
        info_layout.addWidget(self.info_label)
        self.info_frame.setLayout(info_layout)

        # Ana dÃ¼zen
        layout = QVBoxLayout()
        layout.addLayout(button_layout)
        layout.addWidget(self.status_label)
        layout.addLayout(led_layout)
        layout.addWidget(self.info_frame)
        self.setLayout(layout)

        self.device_reachable = False

        # Ana kontrol dÃ¶ngÃ¼sÃ¼ (100 ms)
        self.timer = QTimer()
        self.timer.timeout.connect(self.periodic_tasks)
        self.timer.start(100)

        # Zil dÃ¶ngÃ¼sÃ¼ (2 s)
        self.sound_timer = QTimer()
        self.sound_timer.timeout.connect(self.play_sound)
        self.sound_timer.setInterval(2000)

        self.check_connection()

    def turn_on_output(self):
        if self.device_reachable:
            self.client.write_coil(COIL_ADDRESS, True)

    def turn_off_output(self):
        if self.device_reachable:
            self.client.write_coil(COIL_ADDRESS, False)

    def check_connection(self):
        if ping_device(PLC_IP):
            if not self.device_reachable:
                self.device_reachable = True
                self.client.connect()
            self.status_label.setText("âœ… PLC Ä°le HaberleÅŸme OK..")
            self.status_label.setStyleSheet(
                "color: green; font-size: 14px; font-weight: bold;"
            )
        else:
            self.device_reachable = False
            self.status_label.setText("âŒ PLC HaberleÅŸme HatasÄ±")
            self.status_label.setStyleSheet(
                "color: red; font-size: 14px; font-weight: bold;"
            )

    def check_zil_coil(self):
        if self.device_reachable:
            result = self.client.read_coils(address=ZIL_COIL_ADDRESS, count=1)
            if result is None or result.isError():
                return
            value = result.bits[0]
            if value:
                self.led_indicator.setStyleSheet(
                    "background-color: #90ee90; "
                    "border-radius: 10px; "
                    "border: 1px solid black;"
                )
                if not self.sound_looping:
                    self.sound_timer.start()
                    self.sound_looping = True
            else:
                self.led_indicator.setStyleSheet(
                    "background-color: red; "
                    "border-radius: 10px; "
                    "border: 1px solid black;"
                )
                if self.sound_looping:
                    self.sound_timer.stop()
                    self.sound_looping = False

    def check_1280_led(self):
        if self.device_reachable:
            result = self.client.read_coils(address=LED1280_ADDRESS, count=1)
            if result is None or result.isError():
                return
            value = result.bits[0]
            if value:
                self.led1280.setStyleSheet(
                    "background-color: #90ee90; "
                    "border-radius: 15px; "
                    "border: 2px solid black;"
                )
            else:
                self.led1280.setStyleSheet(
                    "background-color: red; "
                    "border-radius: 15px; "
                    "border: 2px solid black;"
                )

    def play_sound(self):
        QSound.play("ding.wav")

    def periodic_tasks(self):
        self.check_connection()
        self.check_zil_coil()
        self.check_1280_led()

    def closeEvent(self, event):
        self.sound_timer.stop()
        self.client.close()
        event.accept()

if __name__ == "__main__":
    app = QApplication(sys.argv)
    window = ModbusApp()
    window.show()
    sys.exit(app.exec_())
