# Pico-W-Weather-Station
import time
import socket
import network
from machine import Pin, I2C
import bme280

# Wi-Fi credentials
SSID = "    "
PASSWORD = "    "

# Function to connect to Wi-Fi
def connect_to_wifi():
    wlan = network.WLAN(network.STA_IF)
    wlan.active(True)
    wlan.connect(SSID, PASSWORD)
    print(f"Connecting to {SSID}...")
    timeout = 10  # 10 seconds timeout
    for _ in range(timeout * 10):
        if wlan.isconnected():
            ip = wlan.ifconfig()[0]
            print(f"Connected with IP: {ip}")
            return ip
        time.sleep(0.1)
    raise RuntimeError("Failed to connect to Wi-Fi")

# Initialize I2C and BME280
i2c = I2C(0, sda=Pin(0), scl=Pin(1), freq=100000)
bme = bme280.BME280(i2c=i2c, address=0x77)

# Function to generate the webpage content
def serve(connection, bme):
    while True:
        try:
            # Accept a connection from the client
            client, addr = connection.accept()
            print('Client connected from', addr)

            # Read the HTTP request
            request = client.recv(1024)
            print("Request received:\n", request.decode('utf-8'))

            # Ignore favicon requests
            if b"GET /favicon.ico" in request:
                print("Favicon request ignored")
                client.close()
                continue

            # Read sensor data
            temp_celsius = float(bme.values[0][:-1])  # Remove "C" and convert to float
            temp_fahrenheit = (temp_celsius * 9 / 5) + 32
            pressure = bme.values[1]
            humidity = bme.values[2]
            reading = {
                "temp_celsius": f"{temp_celsius:.2f}°C",
                "temp_fahrenheit": f"{temp_fahrenheit:.2f}°F",
                "pressure": pressure,
                "humidity": humidity
            }

            # Generate the HTTP response
            response = webpage(reading)  # Pass reading to the webpage function
            print("Full HTTP Response:\n", response)  # Print full website content to IDE

            # Send the HTTP response and close the socket
            client.sendall(response.encode('utf-8'))
            client.close()
            print("Response sent and client connection closed.")
        except Exception as e:
            print(f"Error during socket communication: {e}")
            client.close()
            break

# Make sure the webpage function receives the reading data
def webpage(reading):
    return f"""HTTP/1.1 200 OK\r\nContent-Type: text/html; charset=UTF-8\r\nConnection: close\r\n\r\n
    <!DOCTYPE html>
    <html>
    <head>
    <title>Pico W Weather Station</title>
    <meta http-equiv="refresh" content="10">
    <style>
        body {{
            background-color: #4ADEDE; /* Light blue background */
            font-family: Arial, sans-serif;
            color: #000000;
            text-align: center;
            margin: 0;
            padding: 0;
            overflow-x: hidden; /* Prevent horizontal scrolling if clouds overflow */
        }}
        .cloud {{
            position: absolute;
            width: 300px; /* Double the size of the cloud */
            height: 180px; /* Double the size of the cloud */
            background-image: url('https://i.imgur.com/RLvh8sF.png'); /* Cloud image */
            background-size: contain; /* Scale image proportionally */
            background-repeat: no-repeat; /* Prevent tiling */
            opacity: 0.8; /* Slightly transparent for a softer look */
        }}
        .cloud1 {{
            top: 20px;
            left: 10%;
        }}
        .cloud2 {{
            top: 200px;
            left: 25%; 
        }}
        .cloud3 {{
            top: 15px;
            left: 70%;
        }}
        h1 {{
            margin-top: 150px; /* Push content below clouds */
        }}
        p {{
            font-size: 1.2em;
        }}
        /* Styling for the image */
        .weather-image {{
            margin-top: 30px; /* Space above the image */
            margin-bottom: 50px; /* Space below the image */
            position: relative;
            animation: moveImage 10s linear infinite; /* Apply animation for right to left movement */
        }}
        .weather-image img {{
            width: 25%; /* Adjust the size of the image (half its original size) */
            max-width: 250px; /* Limit maximum size */
            height: auto; /* Maintain aspect ratio */
        }}
        
        /* Keyframes for moving image from right to left */
        @keyframes moveImage {{
            0% {{
                left: 100%; /* Start off-screen from the right */
                transform: translateX(100%); /* Ensure the image is off-screen */
            }}
            100% {{
                left: -100%; /* Move the image to the left edge */
                transform: translateX(-100%); /* Move the image off-screen to the left */
            }}
        }}
    </style>
    </head>
    <body>
    <!-- Clouds positioned at the top -->
    <div class="cloud cloud1"></div>
    <div class="cloud cloud2"></div>
    <div class="cloud cloud3"></div>

    <!-- Weather Data -->
    <h1>Pico W Weather Station</h1>
    <p><strong>Temperature:</strong> {reading['temp_celsius']}</p>
    <p><strong>Temperature (Fahrenheit):</strong> {reading['temp_fahrenheit']}</p>
    <p><strong>Pressure:</strong> {reading['pressure']}</p>
    <p><strong>Humidity:</strong> {reading['humidity']}</p>

    <!-- Add the image below the weather readings with movement -->
    <div class="weather-image">
        <img src="https://i.imgur.com/HZPDhH9.jpg" alt="Weather Image" /> <!-- Image moving right to left -->
    </div>
    </body>
    </html>
    """
        
# Function to open the server socket
def open_socket(port=8080):
    addr = socket.getaddrinfo('0.0.0.0', port)[0][-1]
    s = socket.socket()
    s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    s.bind(addr)
    s.listen(1)
    print(f"Listening on 0.0.0.0:{port}")
    return s

# Main function
try:
    ip = connect_to_wifi()
    print(f"Server is accessible at http://{ip}:8080")  # Print full IP and port

    s = open_socket(8080)  # Start listening on port 8080
    serve(s, bme)
except Exception as e:
    print(f"Error: {e}")



