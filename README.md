# RTMP Server for OSEE GoStream Recording

This project provides a simple, self-hosted RTMP server solution using Docker and MediaMTX (formerly `rtsp-simple-server`) to record livestreams directly from an OSEE GoStream device to your PC, without requiring additional software like OBS.

## ✨ Features

-   **Direct Recording**: Capture RTMP streams from your OSEE GoStream directly to local storage.
-   **Dockerized**: Easy setup and deployment using Docker Compose.
-   **Customizable Paths**: Define custom recording paths and formats.
-   **Lightweight**: Efficient resource usage with MediaMTX.

## 🚀 Getting Started

### Prerequisites

Before you begin, ensure you have the following installed on your system:

-   **Git**: For cloning the repository.
-   **Docker Desktop**: Includes Docker Engine and Docker Compose for running the server.

For Windows users, you can install both using `winget`:

```powershell
winget install --id Git.Git -h && winget install --id Docker.DockerDesktop -h
```

### Installation

To set up the RTMP server, follow these steps:

1.  **Clone the Repository**:

```powershell
git clone https://github.com/ManiacBoy777/rtmp-server.git
cd rtmp-server
docker compose up -d
```

### Configuration

-   `mediamtx.yml`: Configures the MediaMTX server, including recording settings. You can modify `mediamtx.yml` to change the recording path, format, or other MediaMTX settings as needed. See mediamtx documentation for more information.

**Recording Path**: By default, recorded within your user's `Videos` directory depending on what you choose as a stream key.(e.g., `C:\Users\YourUser\Videos\Livestream`)

## 📺 Usage with OSEE GoStream

Follow these steps to configure your OSEE GoStream to stream to your newly set up RTMP server:

1.  **Open GoStream Control Software**: Launch the `GoStream Control` application on your PC.

2.  **Create Stream XML File**:
    *   Navigate to `Stream > Create Stream XML File`.
    *   For `Platform Name`, enter a descriptive name like `PC Recording`.
    *   For `URL`, enter `rtmp://your-computers-ip-address:1935/live`.
        *   **Note**: To find your computer's IP address, open your command prompt or terminal and type `ipconfig`
    *   Click `EXPORT FILE`.

3.  **Open Streaming Dialogue**:
    *   Go to `Stream > Open Streaming Dialogue`.
    *   Select one of the three available stream slots.
    *   Under `Platform`, choose the platform you just created (e.g., `PC Recording`).
    *   In the `Stream Key` field, enter the desired subfolder name for your recordings (e.g., `my_event`). This will create a folder like `Videos/my_event`.
    *   Ensure `Enable Live` is checked.

Now, when you start streaming from your OSEE GoStream, it will automatically record the live feed as an MP4 file in the specified directory on your PC.

## 🛑 Stopping the Server

To stop the RTMP server and its associated Docker containers, navigate to the `rtmp-server` directory and run:

```powershell
docker compose down
```

## 🤝 Contributing

Contributions are welcome! If you have suggestions for improvements or new features, please open an issue or submit a pull request.

## 📄 License

This project is open-sourced under the MIT License. See the `LICENSE` file for more details.

## 👤 Author

-   **ManiacBoy777**

---
