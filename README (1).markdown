# MCP-Based Web Search & Load RAG System

![Project Status](https://img.shields.io/badge/status-active-brightgreen) ![Python](https://img.shields.io/badge/python-3.8%2B-blue) ![License](https://img.shields.io/badge/license-MIT-blue)

A Retrieval-Augmented Generation (RAG) system that performs web searches using the Google Serper API, loads and indexes documents from trusted sources, and generates accurate answers to user queries. The system uses an MCP (Multi-Cloud Platform) server with `aiohttp` and `ngrok` for external tool integration, built with LangChain 0.3 for modularity and scalability. Designed for Google Colab, it answers domain-specific questions (e.g., "How to prevent Rabies?") by leveraging high-quality sources like `sciencedirect.com` and `frontiersin.org`.

## Table of Contents
- [Features](#features)
- [System Architecture](#system-architecture)
- [Workflow](#workflow)
- [Prerequisites](#prerequisites)
- [Setup](#setup)
- [Usage](#usage)
- [Testing](#testing)
- [Troubleshooting](#troubleshooting)
- [Contributing](#contributing)
- [License](#license)

## Features
- **Web Search**: Performs site-specific searches using the Google Serper API (e.g., `site:aalas.org How to prevent Rabies?`).
- **Document Processing**: Scrapes web pages with `UnstructuredURLLoader`, chunks documents, and indexes them in a FAISS vector store with `HuggingFaceEmbeddings`.
- **RAG Pipeline**: Combines Maximum Marginal Relevance (MMR) retrieval and `gpt-4o-mini` LLM for accurate, context-aware answers.
- **MCP Server**: Runs an `aiohttp` server on port 5555, exposed via `ngrok`, supporting JSON-RPC requests for tool schema and search execution.
- **Robustness**: Includes error handling, retries for HTTP requests, and a fallback mechanism for direct search if the MCP tool fails.
- **Colab-Friendly**: Optimized for Google Colab with API key management via secrets.

## System Architecture
The system follows a modular RAG architecture, integrating an MCP server for external tool access.

```mermaid
graph TD
    A[User Query] --> B[MCP Server<br>(aiohttp, port 5555)]
    B -->|JSON-RPC| C[Google Serper API<br>(site-specific search)]
    C --> D[URLs (max 20)]
    D --> E[UnstructuredURLLoader<br>(web scraping)]
    E --> F[RecursiveCharacterTextSplitter<br>(chunk_size=1000)]
    F --> G[FAISS Vector Store<br>(HuggingFaceEmbeddings)]
    G --> H[MMR Retriever<br>(lambda_mult=0.3)]
    A --> H
    H --> I[ChatPromptTemplate]
    I --> J[ChatOpenAI<br>(gpt-4o-mini)]
    J --> K[StrOutputParser]
    K --> L[Answer]
    B -->|ngrok| M[Public URL]
```

## Workflow
The pipeline processes queries as follows:

```mermaid
graph TD
    A[Start] --> B[Initialize MCP Server]
    B --> C[Expose via ngrok]
    C --> D[Load MCP Tool<br>(SimpleTool)]
    D --> E[Generate Queries<br>(site:domain + question)]
    E --> F[Search URLs<br>(Google Serper API)]
    F --> G{URLs Found?}
    G -->|Yes| H[Limit to 20 URLs]
    G -->|No| I[Retry or Fallback]
    H --> J[Load Documents<br>(UnstructuredURLLoader)]
    J --> K[Split Documents<br>(chunk_size=1000, overlap=200)]
    K --> L[Index in FAISS<br>(all-MiniLM-L6-v2)]
    L --> M[Retrieve Documents<br>(MMR, lambda_mult=0.3)]
    M --> N[Generate Answer<br>(gpt-4o-mini)]
    N --> O[Output Answer]
    O --> P[Disconnect ngrok]
    P --> Q[End]
```

## Prerequisites
- **Python**: 3.8 or higher
- **Google Colab**: For running the notebook
- **API Keys**:
  - [OpenAI API Key](https://openai.com) (`OPENAI_API_KEY4`)
  - [Google Serper API Key](https://serper.dev) (`SERPER_API_KEY`)
  - [ngrok Authtoken](https://ngrok.com) (`NGROK_TOKEN`)
- **Dependencies**:
  ```bash
  langchain==0.3.0
  langchain-community
  langchain-openai
  langchain-mcp-adapters
  faiss-cpu
  beautifulsoup4
  unstructured
  sentence-transformers
  aiohttp
  pyngrok
  ```

## Setup
1. **Clone the Repository** (if hosted on GitHub):
   ```bash
   git clone https://github.com/your-username/mcp-rag-system.git
   cd mcp-rag-system
   ```
2. **Set Up Google Colab**:
   - Open the notebook in Colab: [MCP-Based-WebSearch&LoadRAG.ipynb](https://colab.research.google.com/drive/16iIIfBvG9f5ZbAfa4UWK1V2wEGoFDnLb).
   - Add API keys to Colab secrets:
     - `OPENAI_API_KEY4`: Your OpenAI API key.
     - `SERPER_API_KEY`: Your Google Serper API key.
     - `NGROK_TOKEN`: Your ngrok authtoken.
3. **Install Dependencies**:
   Run the following in a Colab cell:
   ```bash
   !pip install --upgrade -q langchain langchain-community langchain-openai langchain-mcp-adapters faiss-cpu beautifulsoup4 unstructured sentence-transformers aiohttp pyngrok
   ```

## Usage
1. **Run the Notebook**:
   - Execute all cells in the Colab notebook.
   - The MCP server starts on port 5555 and is exposed via `ngrok` (e.g., `https://<random>.ngrok-free.app`).
   - The system fetches URLs for the query "How to prevent Rabies?" from trusted domains, processes documents, and generates an answer.
2. **Example Output**:
   ```
   MCP server running on port 5555
   MCP server exposed at: https://<random>.ngrok-free.app
   Loaded tool schema: {'name': 'google_serper_search', ...}
   Loaded tools: ['google_serper_search']
   Searching URLs: 100%|██████████| 28/28 [00:15<00:00,  1.80it/s]
   Total unique article URLs fetched: 12
   Total chunks: 45
   Answer:
   To prevent rabies, vaccinate pets regularly, avoid contact with wild animals, and seek immediate medical attention if bitten. Pre-exposure prophylaxis is recommended for high-risk individuals. [Source: cdc.gov, who.int]
   ```

## Testing
To test the `google_serper_search` function:
1. Uncomment the `test_search_function` in the code.
2. Run the cell:
   ```python
   await test_search_function()
   ```
3. Example Output:
   ```
   Testing google_serper_search function:
   Searching for: aalas.org: How to prevent Rabies?
   Results:
     Title: Rabies Prevention in Laboratory Animals
     Link: https://www.aalas.org/about-aalas/rabies-prevention
   ```

## Troubleshooting
- **No URLs Fetched**:
  - Verify `SERPER_API_KEY` is valid in Colab secrets.
  - Run `test_search_function` to debug specific domains.
  - Some domains (e.g., `jvas.in`) may return no results due to limited content.
- **ngrok Errors**:
  - Free-tier sessions expire after 2 hours. Restart the cell to get a new URL.
  - Check `NGROK_TOKEN` in Colab secrets.
- **Port Conflicts**:
  - Change the port (e.g., 5556) in `start_server` and `ngrok.connect` if 5555 is in use.
- **Document Loading Issues**:
  - Add `SeleniumURLLoader` for JS-heavy websites (requires additional setup).
  - Check URLs for accessibility or paywalls.

## Contributing
Contributions are welcome! To contribute:
1. Fork the repository.
2. Create a new branch (`git checkout -b feature/your-feature`).
3. Make changes and commit (`git commit -m "Add your feature"`).
4. Push to the branch (`git push origin feature/your-feature`).
5. Open a pull request.

Please include a `CONTRIBUTING.md` file or follow the guidelines in [CONTRIBUTING.md](CONTRIBUTING.md) if available.[](https://www.makeareadme.com/)

## License
This project is licensed under the MIT License. See [LICENSE](LICENSE) for details.