******************
Using the compiler
******************

.. index:: ! commandline compiler, compiler;commandline, ! solc, ! linker

.. _commandline-compiler:

Using the Commandline Compiler
******************************

One of the build targets of the Solidity repository is ``solc``, the solidity commandline compiler.
Using ``solc --help`` provides you with an explanation of all options. The compiler can produce various outputs, ranging from simple binaries and assembly over an abstract syntax tree (parse tree) to estimations of gas usage.
If you only want to compile a single file, you run it as ``solc --bin sourceFile.sol`` and it will print the binary. Before you deploy your contract, activate the optimizer while compiling using ``solc --optimize --bin sourceFile.sol``. If you want to get some of the more advanced output variants of ``solc``, it is probably better to tell it to output everything to separate files using ``solc -o outputDirectory --bin --ast --asm sourceFile.sol``.

The commandline compiler will automatically read imported files from the filesystem, but
it is also possible to provide path redirects using ``prefix=path`` in the following way:

::

    solc github.com/ethereum/dapp-bin/=/usr/local/lib/dapp-bin/ =/usr/local/lib/fallback file.sol

This essentially instructs the compiler to search for anything starting with
``github.com/ethereum/dapp-bin/`` under ``/usr/local/lib/dapp-bin`` and if it does not
find the file there, it will look at ``/usr/local/lib/fallback`` (the empty prefix
always matches). ``solc`` will not read files from the filesystem that lie outside of
the remapping targets and outside of the directories where explicitly specified source
files reside, so things like ``import "/etc/passwd";`` only work if you add ``=/`` as a remapping.

If there are multiple matches due to remappings, the one with the longest common prefix is selected.

For security reasons the compiler has restrictions what directories it can access. Paths (and their subdirectories) of source files specified on the commandline and paths defined by remappings are allowed for import statements, but everything else is rejected. Additional paths (and their subdirectories) can be allowed via the ``--allow-paths /sample/path,/another/sample/path`` switch.

If your contracts use :ref:`libraries <libraries>`, you will notice that the bytecode contains substrings of the form ``__LibraryName______``. You can use ``solc`` as a linker meaning that it will insert the library addresses for you at those points:

Either add ``--libraries "Math:0x12345678901234567890 Heap:0xabcdef0123456"`` to your command to provide an address for each library or store the string in a file (one library per line) and run ``solc`` using ``--libraries fileName``.

If ``solc`` is called with the option ``--link``, all input files are interpreted to be unlinked binaries (hex-encoded) in the ``__LibraryName____``-format given above and are linked in-place (if the input is read from stdin, it is written to stdout). All options except ``--libraries`` are ignored (including ``-o``) in this case.

If ``solc`` is called with the option ``--standard-json``, it will expect a JSON input (as explained below) on the standard input, and return a JSON output on the standard output.

.. _compiler-api:

Compiler Input and Output JSON Description
******************************************

These JSON formats are used by the compiler API as well as are available through ``solc``. These are subject to change,
some fields are optional (as noted), but it is aimed at to only make backwards compatible changes.

The compiler API expects a JSON formatted input and outputs the compilation result in a JSON formatted output.

Comments are of course not permitted and used here only for explanatory purposes.

Input Description
-----------------

.. code-block:: none

    {
      // Requerido: Lenguaje del código fuente, tal como "Solidity", "serpent", "lll", "assembly", etc.
      language: "Solidity",
      // Requerido
      sources:
      {
        // Las teclas aquí son los nombres "globales" de los ficheros fuente,
        // las importaciones pueden utilizar otros ficheros mediante remappings (vér más abajo).
        "myFile.sol":
        {
          // Opcional: keccak256 hash del fichero fuente
          // Se utiliza para verificar el contenido recuperado si se importa a través de URLs.
          "keccak256": "0x123...",
          // Requerido (a menos que se use "contenido", ver abajo): URL (s) al fichero fuente.
          // URL(s) deben ser importadas en este orden y el resultado debe ser verificado contra el fichero
          // keccak256 hash (si está disponible). Si el hash no coincide o no coincide con ninguno de los
          // URL(s) resultado en el éxito, un error debe ser elevado.
          "urls":
          [
            "bzzr://56ab...",
            "ipfs://Qma...",
            "file:///tmp/path/to/file.sol"
          ]
        },
        "mortal":
        {
          // Opcional: keccak256 hash del fichero fuente
          "keccak256": "0x234...",
          // Requerido (a menos que se use "urls"): contenido literal del fichero fuente
          "content": "contract mortal is owned { function kill() { if (msg.sender == owner) selfdestruct(owner); } }"
        }
      },
      // Opcional
      settings:
      {
        // Opcional: Lista ordenada de remappings
        remappings: [ ":g/dir" ],
        // Opcional: Ajustes de optimización (activación de valores predeterminados a false)
        optimizador: {
          enabled: true,
          runs: 500
        },
        // Configuración de metadatos (opcional)
        metadata: {
          // Usar sólo contenido literal y no URLs (falso por defecto)
          useLiteralContent: true
        },
        // Direcciones de las bibliotecas. Si no todas las bibliotecas se dan aquí, puede resultar con objetos no vinculados cuyos datos de salida son diferentes.
        libraries: {
          // La clave superior es el nombre del fichero fuente donde se utiliza la biblioteca.
          // Si se utiliza remappings, este fichero fuente debe coincidir con la ruta global después de que se hayan aplicado los remappings.
          // Si esta clave es una cadena vacía, se refiere a un nivel global.

          "myFile.sol": {
            "MyLib": "0x123123..."
          }
        }
        // Para seleccionar las salidas deseadas se puede utilizar lo siguiente.
        // Si este campo se omite, el compilador se carga y comprueba el tipo, pero no genera ninguna salida aparte de errores.
        // La clave de primer nivel es el nombre del fichero y la segunda es el nombre del contrato, donde el nombre vacío del contrato se refiere al fichero mismo,
        // mientras que la estrella se refiere a todos los contratos.
        //
        // Las clases de mensajes disponibles son las siguientes:
        //   abi - ABI
        //   ast - AST de todos los ficheros fuente
        //   legacyAST - Legado AST de todos los ficheros fuente
        //   devdoc - Documentación para desarrolladores (natspec)
        //   userdoc - Documentación de usuario (natspec)
        //   metadata - Metadatos
        //   ir - Nuevo formato de montaje antes del desazucarado
        //   evm.assembly - Nuevo formato de montaje después del desazucarado
        //   evm.legacyAssembly - Formato de montaje antiguo en JSON
        //   evm.bytecode.object - Objeto bytecode
        //   evm.bytecode.opcodes - Lista de Opcodes
        //   evm.bytecode.sourceMap - Asignación de fuentes (útil para depuración)
        //   evm.bytecode.linkReferences - Referencias de enlace (si es objeto no enlazado)
        //   evm.deployedBytecode* - Desplegado bytecode (tiene las mismas opciones que evm.bytecode)
        //   evm.methodIdentifiers - La lista de funciones de hashes 
        //   evm.gasEstimates - Funcion de estimación de gas
        //   ewasm.wast - eWASM S-formato de expresiones (no compatible con atm)
        //   ewasm.wasm - eWASM formato binario (no compatible con atm)
        //
        // Ten en cuenta que el uso de `evm`, `evm.bytecode`, `ewasm`, etc. seleccionara cada
        // parte objetiva de esa salida.
        //
        outputSelection: {
          // Habilita los metadatos y las salidas de bytecode de cada contrato.
          "*": {
            "*": [ "metadata", "evm.bytecode" ]
          },
          // Habilitar la salida abi y opcodes de MyContract definida en el fichero def.
          "def": {
            "MyContract": [ "abi", "evm.opcodes" ]
          },
          // Habilita la salida del mapa de fuentes de cada contrato individual.
          "*": {
            "*": [ "evm.sourceMap" ]
          },
          // Habilita la salida AST heredada de cada archivo.
          "*": {
            "": [ "legacyAST" ]
          }
        }
      }
    }


Output Description
------------------

.. code-block:: none

    {
      // Optional: not present if no errors/warnings were encountered
      errors: [
        {
          // Optional: Location within the source file.
          sourceLocation: {
            file: "sourceFile.sol",
            start: 0,
            end: 100
          ],
          // Mandatory: Error type, such as "TypeError", "InternalCompilerError", "Exception", etc
          type: "TypeError",
          // Mandatory: Component where the error originated, such as "general", "ewasm", etc.
          component: "general",
          // Mandatory ("error" or "warning")
          severity: "error",
          // Mandatory
          message: "Invalid keyword"
          // Optional: the message formatted with source location
          formattedMessage: "sourceFile.sol:100: Invalid keyword"
        }
      ],
      // This contains the file-level outputs. In can be limited/filtered by the outputSelection settings.
      sources: {
        "sourceFile.sol": {
          // Identifier (used in source maps)
          id: 1,
          // The AST object
          ast: {},
          // The legacy AST object
          legacyAST: {}
        }
      },
      // This contains the contract-level outputs. It can be limited/filtered by the outputSelection settings.
      contracts: {
        "sourceFile.sol": {
          // If the language used has no contract names, this field should equal to an empty string.
          "ContractName": {
            // The Ethereum Contract ABI. If empty, it is represented as an empty array.
            // See https://github.com/ethereum/wiki/wiki/Ethereum-Contract-ABI
            abi: [],
            // See the Metadata Output documentation (serialised JSON string)
            metadata: "{...}",
            // User documentation (natspec)
            userdoc: {},
            // Developer documentation (natspec)
            devdoc: {},
            // Intermediate representation (string)
            ir: "",
            // EVM-related outputs
            evm: {
              // Assembly (string)
              assembly: "",
              // Old-style assembly (object)
              legacyAssembly: {},
              // Bytecode and related details.
              bytecode: {
                // The bytecode as a hex string.
                object: "00fe",
                // Opcodes list (string)
                opcodes: "",
                // The source mapping as a string. See the source mapping definition.
                sourceMap: "",
                // If given, this is an unlinked object.
                linkReferences: {
                  "libraryFile.sol": {
                    // Byte offsets into the bytecode. Linking replaces the 20 bytes located there.
                    "Library1": [
                      { start: 0, length: 20 },
                      { start: 200, length: 20 }
                    ]
                  }
                }
              },
              // The same layout as above.
              deployedBytecode: { },
              // The list of function hashes
              methodIdentifiers: {
                "delegate(address)": "5c19a95c"
              },
              // Function gas estimates
              gasEstimates: {
                creation: {
                  codeDepositCost: "420000",
                  executionCost: "infinite",
                  totalCost: "infinite"
                },
                external: {
                  "delegate(address)": "25000"
                },
                internal: {
                  "heavyLifting()": "infinite"
                }
              }
            },
            // eWASM related outputs
            ewasm: {
              // S-expressions format
              wast: "",
              // Binary format (hex string)
              wasm: ""
            }
          }
        }
      }
    }
