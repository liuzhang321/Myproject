#!/usr/bin/env python3
import sys
import signal
import os
import time
import serial
import speech_recognition as sr
import requests
from gtts import gTTS
import json
from geopy.geocoders import Nominatim

# GraphHopper配置
GRAPHOPPER_URL = "http://localhost:8989/route"
DOCKER_CONTAINER_NAME = "graphhopper"

# 串口配置
UART_DEV = '/dev/ttyS1'
BAUDRATE = 9600
SERIAL_TIMEOUT = 1

# 全局变量存储位置数据
current_position = None
ser = None
position_received = False  # 新增：位置接收状态标志


def signal_handler(signal, frame):
    """处理CTRL+C信号"""
    global ser
    if ser and ser.is_open:
        ser.close()
        print("Serial port closed.")
    sys.exit(0)


def parse_position_data(data):
    """解析格式为: b'N031.22296E120.57951' 的定位数据"""
    try:
        # 解码数据
        data_str = data.decode('utf-8', errors='ignore').strip()
        print(f"接收数据: {data_str}")  # 调试信息

        # 检查格式是否符合: N开头，E在中间
        if not (data_str.startswith('N') and 'E' in data_str):
            print(f"数据格式错误: {data_str}")
            return None

        # 找到E的位置
        e_index = data_str.index('E')

        # 提取纬度 (N后的部分到E之前)
        lat_str = data_str[1:e_index]
        # 提取经度 (E后的部分)
        lng_str = data_str[e_index + 1:]

        # 验证纬度格式 (应包含小数点)
        if '.' not in lat_str:
            print(f"纬度格式错误: {lat_str}")
            return None

        # 验证经度格式 (应包含小数点)
        if '.' not in lng_str:
            print(f"经度格式错误: {lng_str}")
            return None

        # 提取有效数字部分
        try:
            # 纬度: 确保有小数点前3位，后5位
            lat_parts = lat_str.split('.')
            if len(lat_parts) != 2 or len(lat_parts[0]) < 2 or len(lat_parts[1]) < 5:
                print(f"纬度格式无效: {lat_str}")
                return None
            lat = float(f"{lat_parts[0][:2]}.{lat_parts[1][:5]}")

            # 经度: 确保有小数点前3位，后5位
            lng_parts = lng_str.split('.')
            if len(lng_parts) != 2 or len(lng_parts[0]) < 3 or len(lng_parts[1]) < 5:
                print(f"经度格式无效: {lng_str}")
                return None
            lng = float(f"{lng_parts[0][:3]}.{lng_parts[1][:5]}")

            return (lat, lng)

        except (ValueError, IndexError) as e:
            print(f"数值转换错误: {e}")
            return None

    except Exception as e:
        print(f"解析异常: {e}")
        return None


def receive_position_data():
    """接收并解析STM32发送的定位数据"""
    global ser, current_position, position_received

    try:
        # 打开串口 - 确保参数完整
        ser = serial.Serial(
            port=UART_DEV,
            baudrate=BAUDRATE,
            bytesize=8,
            parity='N',
            stopbits=1,
            timeout=SERIAL_TIMEOUT
        )
        print(f"串口已打开: {ser}")

        print("等待接收定位数据... 按CTRL+C退出")

        # 修改：添加超时机制和位置接收标志
        start_time = time.time()
        timeout = 30  # 30秒超时

        while not position_received and (time.time() - start_time < timeout):
            try:
                # 使用readline()读取数据
                received_data = ser.readline()

                if received_data:
                    # 显示原始数据（十六进制格式）
                    print("原始数据:", received_data.hex())

                    # 解析位置数据
                    position = parse_position_data(received_data)

                    if position:
                        current_position = position
                        position_received = True  # 设置标志位
                        print(f"当前位置: 纬度={position[0]}, 经度={position[1]}")
                else:
                    # 无数据时短暂等待
                    time.sleep(0.1)

            except (UnicodeDecodeError, serial.SerialException) as e:
                print(f"通信错误: {e}")
            except KeyboardInterrupt:
                print("\n用户终止程序")
                break  # 退出循环

        # 关闭串口，不再需要持续接收
        if ser and ser.is_open:
            ser.close()
            print("串口已关闭.")

        return 0 if position_received else -1

    except serial.SerialException as e:
        print(f"打开串口失败! 错误: {e}")
        return -1


def recognize_chinese_speech():
    """使用麦克风录制并识别中文语音"""
    r = sr.Recognizer()
    with sr.Microphone() as source:
        print("请说出终点地址...")
        r.adjust_for_ambient_noise(source, duration=1)
        audio = r.listen(source, timeout=10)

    try:
        end_text = r.recognize_google(audio, language='zh-CN')
        print(f"识别终点: {end_text}")
        return end_text.strip()

    except sr.UnknownValueError:
        print("抱歉，我没有听清楚")
        return None
    except sr.RequestError:
        print("语音服务不可用")
        return None


def geocode_address(address):
    """将中文地址转换为经纬度坐标"""
    geolocator = Nominatim(user_agent="graphhopper_navigation")
    try:
        location = geolocator.geocode(address + ", 中国", timeout=10)
        if location:
            print(f"地址解析成功: {location.address}")
            return (location.latitude, location.longitude)
    except Exception as e:
        print(f"地理编码错误: {e}")
    return None


def get_route(start_coord, end_coord):
    """调用GraphHopper API获取路线"""
    params = {
        'point': [f"{start_coord[0]},{start_coord[1]}",
                  f"{end_coord[0]},{end_coord[1]}"],
        'vehicle': 'car',
        'locale': 'zh_CN',
        'points_encoded': False,
        'instructions': True,
        'key': ''
    }

    try:
        response = requests.get(GRAPHOPPER_URL, params=params, timeout=15)
        if response.status_code == 200:
            return response.json()
        print(f"API错误: {response.status_code}")
        return None
    except requests.RequestException as e:
        print(f"连接GraphHopper失败: {e}")
        return None


def parse_route(route_data):
    """解析路线结果生成中文导航指令"""
    if not route_data or 'paths' not in route_data or not route_data['paths']:
        return "未找到路线"

    path = route_data['paths'][0]
    distance = path['distance'] / 1000  # 转换为公里
    time_min = path['time'] / 60000  # 转换为分钟

    instructions = []
    for instr in path['instructions']:
        # 简化指令文本
        text = instr['text'].replace("Continue onto ", "继续沿")
        text = text.replace("Turn left onto ", "左转进入")
        text = text.replace("Turn right onto ", "右转进入")
        text = text.replace(" at ", "在")
        instructions.append(f"{text} ({instr['distance']}米)")

    return {
        'summary': f"总距离: {distance:.1f}公里, 预计时间: {time_min:.0f}分钟",
        'instructions': instructions
    }


def speak(text, lang='zh-CN'):
    """语音播报文本"""
    print(f"语音播报: {text}")
    try:
        tts = gTTS(text=text, lang=lang, slow=False)
        tts.save("nav_output.mp3")
        # 修改：指定使用card 0设备播放
        os.system("mpg321 -q -a hw:0,0 nav_output.mp3")  # 使用card 0设备
        os.remove("nav_output.mp3")
    except Exception as e:
        print(f"语音合成失败: {e}")


def main_navigation():
    """主导航流程"""
    # 检查Docker容器是否运行
    if os.system(f"docker ps | grep -q {DOCKER_CONTAINER_NAME}") != 0:
        speak("GraphHopper容器未运行，请先启动容器")
        return

    # 获取当前位置
    if not current_position:
        print("未获取到当前位置信息")
        return

    # 语音输入终点
    end_addr = recognize_chinese_speech()
    if not end_addr:
        return

    # 地址解析
    end_coord = geocode_address(end_addr)
    if not end_coord:
        speak(f"无法解析地址: {end_addr}")
        return

    # 获取路线
    route_data = get_route(current_position, end_coord)

    # 解析结果
    if not route_data:
        speak("路线规划失败")
        return

    result = parse_route(route_data)
    speak(result['summary'])

    # 播报详细指引
    for i, instruction in enumerate(result['instructions']):
        speak(f"第{i + 1}步: {instruction}")
        time.sleep(1.5)  # 步骤间暂停


if __name__ == '__main__':
    # 注册信号处理
    signal.signal(signal.SIGINT, signal_handler)

    print("=== STM32定位导航系统 ===")

    # 步骤1: 接收STM32定位数据
    if receive_position_data() == -1:
        print("串口测试失败!")
        sys.exit(1)

    # 步骤2: 进行导航
    main_navigation()

    print("导航完成!")
