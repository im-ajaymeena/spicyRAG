### Start the postgres database 
1. `cd ./postgres`

2. `docker compose up -d`


### Add file .env with

``` 
OPENAI_API_KEY=sk-Vs3f..
ANTHROPIC_API_KEY=sk-ant..
DEEPGRAM_API_KEY=43af..
```


### Architecture
#### Index creation
```mermaid
flowchart TD
subgraph above["Data Loading and indexing"]
 subgraph subGraph0["Create Text Nodes For each File"]
        E["Concatenate Transcript"]
        F["Extract Recipe Title"]
        G["Create list of File Nodes"]
        FF["Title Question Extractor (Metadata)"]
  end

 subgraph subGraph1["Create Text Nodes For each Paragraph"]
        I["Get Paragraph text (5 consequtive sentences)"]
        J["Extract Food Entity"]
        H["Create Paragraph Nodes: pnodes with metadata <br> start_time, end_time, filehash"]
  end
    II["ffmpeg commands for extracting pic_type frame corresponding to <br> each node, save extracted frames inside dir file_hash/node_hash <br> with filename as relative frame_number"]

    B["File Transcript List"]
    B --> C{"For Each File Transcript"}
    C -->   I &  D["Extract File Metadata"]
    
    I --> J
    HI["Create Vectore Store Index: pindex"]
    G --> HI["Create Vectore Store Index: file_index"]
    D -->  E
    E --> F
    F --> FF
    FF --> G 

    J  --> H

    H --> II
    II --> IJ["Create Image nodes with extracted frames"]
    IJ --> JK['create Multi-Modal index with PGStore']
    style subGraph0 stroke:#000000,fill:none
    style subGraph1 stroke:#000000,fill:none
end
```

#### Response generation
```mermaid
flowchart TB
    Start(["User submit query"]) --> GenerateRecipe{"OpenAI Agent for function selection\n(User Query)"}
        GenerateRecipe -->|Custom Dish Recipe| CDS["Sample 100 Paragraph Nodes"]
        GenerateRecipe -->|Known Dish Recipe| DOC["Fetch relavant File containing Recipe"]
        GenerateRecipe -->|Recipe| KOK["Fetch doc_node corresponding to Recipe"]
        GenerateRecipe -->|Modification request| PPK[Modification Request]
    subgraph subGraph0["Generate Modified Recipe "]
        KOK --> MN["Retrive nodes relevant to recipe in query"]
         
        MN --> NUN
        PPK --> NUN
    end
    subgraph subGraph2["Unknown Recipe generate"]
        CDS --> HUH["Create Retriever corresponding to nodes"]

    end
    subgraph subGraph1["Summarize known recipe in markdown format"]
        DOC --> PP{"Similarity > thres"}
        PP -->|yes| KDR["pindex -> query-engine -> get-recipe summary"]
        PP -->|no| KDP["return 'could not find user query'"]
        KDR --> JJ["Retrive relavant source node for each step in recipe"]
        GIS --> RI["Among Retrive images, Retrieve images for recipe steps"]
        RI & KDR --> RMMLM[Refine each step with Multi-Modal LM]
        RMMLM -->|response| MK("Create/Show Markdown (Fetch frame timestamp)")
        JJ --> GIS["For each relevant node -> Get Images all images corresponding to node"]
       
    end
    HUH --> OJ["For each step of recipe generate image <br> and get its url using DALLE"]
    NUN[Response synthesizer] -->|response|OJ
    OJ -->|response| MKK("Create/Show Markdown")

    style subGraph0 stroke:#000000,fill:none
    style subGraph1 stroke:#000000,fill:none
    style subGraph2 stroke:#000000,fill:none
```