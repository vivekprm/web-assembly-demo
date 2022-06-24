# Prerequisites
- Install Go 
- Install tiny go compiler from below link.
https://tinygo.org/getting-started/install/

- Install goexec to start web server to serve web assembly.
```sh
go get -u github.com/shurcooL/goexec
```

# Steps
- Generate web assembly file corresponding to main.go
```sh
tinygo build -o main.wasm -target wasm ./main.go
```
- Copy wasm_exec.js for the same compiler which generated the web assembly.
```sh
cp $(tinygo env TINYGOROOT)/targets/wasm_exec.js .
```
- Create index.js which provides the bridge code to call web assembly.

```js
// https://github.com/torch2424/wasm-by-example/blob/master/demo-util/
export const wasmBrowserInstantiate = async (wasmModuleUrl, importObject) => {
  let response = undefined;

  // Check if the browser supports streaming instantiation
  if (WebAssembly.instantiateStreaming) {
    // Fetch the module, and instantiate it as it is downloading
    response = await WebAssembly.instantiateStreaming(
      fetch(wasmModuleUrl),
      importObject
    );
  } else {
    // Fallback to using fetch to download the entire module
    // And then instantiate the module
    const fetchAndInstantiateTask = async () => {
      const wasmArrayBuffer = await fetch(wasmModuleUrl).then(response =>
        response.arrayBuffer()
      );
      return WebAssembly.instantiate(wasmArrayBuffer, importObject);
    };

    response = await fetchAndInstantiateTask();
  }

  return response;
};

const go = new Go(); // Defined in wasm_exec.js. Don't forget to add this in your index.html.

const runWasmAdd = async () => {
  // Get the importObject from the go instance.
  const importObject = go.importObject;

  // Instantiate our wasm module
  const wasmModule = await wasmBrowserInstantiate("./main.wasm", importObject);

  // Allow the wasm_exec go instance, bootstrap and execute our wasm module
  go.run(wasmModule.instance);

  // Call the Add function export from wasm, save the result
  const addResult = wasmModule.instance.exports.add(24, 24);

  // Set the result onto the body
  document.body.textContent = `Hello World! addResult: ${addResult}`;
};
runWasmAdd();
```

- Lastly to serve these file run web server supporting the web assembly.

```sh
goexec 'http.ListenAndServe(`:8080`, http.FileServer(http.Dir(`.`)))'
```
