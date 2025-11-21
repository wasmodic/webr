# webR

## How does the app work?
Look at the `<script/>` html section. R is started in a few key steps:

1. webR is imported, input and output html chunks are defined, and a new webR instance is made:
    ```javascript
    import { WebR } from "https://webr.r-wasm.org/latest/webr.mjs";

    const statusEl = document.getElementById("status");
    const outputEl = document.getElementById("output");
    const runBtn = document.getElementById("run");

    const webR = new WebR();
    ```

2. Default R code and data is created in in a `webR.evalR()` command. Note that `async` means code returns in a stream while `R` evaluates.
    ```javascript
    await webR.evalR(`
        clean_seq <- function(x) {
        lines <- unlist(strsplit(x, "\\n"))
        seq_lines <- lines[!grepl("^>", lines)]
        seq <- paste(seq_lines, collapse = "")
        seq <- toupper(gsub("[^ACGT]", "", seq))
        seq
        } 
        ... `, { captureStreams: false });
    }
    ```

## How do data enter WASM here?
For the sequence input, they go into a html element defined as `seq`. These are then bound into `R` with `webR.objs.globalEnv.bind()`
    ```javascript
        runBtn.addEventListener("click", async () => {
            runBtn.disabled = true;
            statusEl.textContent = "Running...";

            const seq = document.getElementById("seq").value;

            // Move input text into R
            await webR.objs.globalEnv.bind("rawseq", seq);
    ```
Input is then processed like above in `webR.evalR()`
    ```javascript
      const result = await webR.evalR(`
        library(stringr)

        seq_clean <- clean_seq(rawseq)
        gc <- gc_content(seq_clean)
        revseq <- rev_comp(seq_clean)

        paste(
        sprintf("Cleaned length: %d", gc$len),
        sprintf("GC bases: %d", gc$gc),
        sprintf("GC content: %.2f%%", gc$pct),
        "",
        "Reverse complement:",
        revseq,
        sep = "\\n"
        )
    ...)
    ```

## How do data exist WASM?
R output is captured in the result variable defined above and then handed back to the DOM via:
```javascript
 const text = await result.toJs();
      outputEl.textContent = text.values[0];
      statusEl.textContent = "Done.";
      runBtn.disabled = false;
```

## Questions:
- What does captureStreams mean? Surely it's all a stream if it's async?!
- What does `result.toJS()` do?
- What does `runBtn.disabled = false` mean?