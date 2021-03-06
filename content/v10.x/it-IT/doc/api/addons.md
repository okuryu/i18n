# Addons C++

<!--introduced_in=v0.10.0-->
<!-- type=misc -->

Gli Addons di Node.js sono shared objects con collegamenti dinamici, scritti in C++, che possono essere caricati all'interno di Node.js usando la funzione [`require()`](modules.html#modules_require), ed utilizzati come se fossero un normale modulo di Node.js. Essi vengono utilizzati principalmente per fornire un'interfaccia tra JavaScript in esecuzione in Node.js e le librerie C/C++.

Al momento, il metodo per implementare gli Addons è piuttosto complicato, coinvolgendo la conoscenza di diversi componenti e diversi API:

 - V8: la libreria C++ che Node.js utilizza attualmente per fornire l'implementazione JavaScript. V8 fornisce i meccanismi per creare oggetti, chiamare funzioni, ecc. L'API di V8 è documentata principalmente nel file di intestazione `v8.h` (`deps/v8/include/v8.h` nell'albero sorgente di Node.js), che è anche disponibile [online](https://v8docs.nodesource.com/).

 - [libuv](https://github.com/libuv/libuv): La libreria C che implementa il ciclo di eventi Node.js, i suoi thread di lavoro e tutti i comportamenti asincroni della piattaforma. Inoltre, funge da libreria di astrazione multipiattaforma, dando un facile accesso di tipo POSIX, su tutti i principali sistemi operativi, a molte attività di sistema comuni come l'interazione con il filesystem, i socket, i timer e gli eventi di sistema. libuv fornisce anche un'astrazione di thread di tipo pthreads (POSIX-threads) che può essere usata per alimentare gli Addons asincroni più sofisticati che devono andare oltre il ciclo degli eventi standard. Gli autori degli Addon sono incoraggiati a pensare a come evitare il blocco del ciclo degli eventi con I/O oppure con altre attività che richiedono molto tempo nello scaricare il lavoro tramite libuv in operazioni di sistema non-blocking, threads di lavoro od un uso personalizzato dei threads di libuv.

 - Librerie interne di Node.js. Node.js stesso esporta un numero di API C++ che gli Addons possono utilizzare &mdash; la più importante delle quali è la classe `node::ObjectWrap`.

 - Node.js include una serie di altre librerie collegate in modo statico, tra cui OpenSSL. Queste altre librerie si trovano nella directory `deps/` all'interno dell'albero sorgente (source tree) di Node.js. Solo i simboli di libuv, OpenSSL, V8 e zlib vengono appositamente ri-esportati da Node.js e possono essere utilizzati in varie estensioni dagli Addons. Vedi [Collegamento alle dipendenze di Node.js](#addons_linking_to_node_js_own_dependencies) per ulteriori informazioni.

Tutti i seguenti esempi sono disponibili per il [download](https://github.com/nodejs/node-addon-examples) e possono essere utilizzati come punto di partenza per un Addon.

## Hello world

Questo esempio di "Hello world" è un semplice Addon, scritto in C++, che è l'equivalente del seguente codice JavaScript:

```js
module.exports.hello = () => 'world';
```

Prima di tutto, crea il file `hello.cc`:

```cpp
// hello.cc
#include <node.h>

namespace demo {

using v8::FunctionCallbackInfo;
using v8::Isolate;
using v8::Local;
using v8::NewStringType;
using v8::Object;
using v8::String;
using v8::Value;

void Method(const FunctionCallbackInfo<Value>& args) {
  Isolate* isolate = args.GetIsolate();
  args.GetReturnValue().Set(String::NewFromUtf8(
      isolate, "world", NewStringType::kNormal).ToLocalChecked());
}

void Initialize(Local<Object> exports) {
  NODE_SET_METHOD(exports, "hello", Method);
}

NODE_MODULE(NODE_GYP_MODULE_NAME, Initialize)

}  // namespace demo
```

Si noti che tutti gli Addons di Node.js devono esportare una funzione di inizializzazione seguendo il modello:

```cpp
void Initialize(Local<Object> exports);
NODE_MODULE(NODE_GYP_MODULE_NAME, Initialize)
```

Non c'è alcun punto e virgola dopo `NODE_MODULE` poiché non è una funzione (vedi `node.h`).

Il `module_name` deve corrispondere al filename del file binario finale (escludendo il suffisso `.node`).

In the `hello.cc` example, then, the initialization function is `Initialize` and the addon module name is `addon`.

When building addons with `node-gyp`, using the macro `NODE_GYP_MODULE_NAME` as the first parameter of `NODE_MODULE()` will ensure that the name of the final binary will be passed to `NODE_MODULE()`.

### Context-aware addons

There are environments in which Node.js addons may need to be loaded multiple times in multiple contexts. For example, the [Electron](https://electronjs.org/) runtime runs multiple instances of Node.js in a single process. Each instance will have its own `require()` cache, and thus each instance will need a native addon to behave correctly when loaded via `require()`. From the addon's perspective, this means that it must support multiple initializations.

A context-aware addon can be constructed by using the macro `NODE_MODULE_INITIALIZER`, which expands to the name of a function which Node.js will expect to find when it loads an addon. An addon can thus be initialized as in the following example:

```cpp
using namespace v8;

extern "C" NODE_MODULE_EXPORT void
NODE_MODULE_INITIALIZER(Local<Object> exports,
                        Local<Value> module,
                        Local<Context> context) {
  /* Perform addon initialization steps here. */
}
```

Another option is to use the macro `NODE_MODULE_INIT()`, which will also construct a context-aware addon. Unlike `NODE_MODULE()`, which is used to construct an addon around a given addon initializer function, `NODE_MODULE_INIT()` serves as the declaration of such an initializer to be followed by a function body.

The following three variables may be used inside the function body following an invocation of `NODE_MODULE_INIT()`:
* `Local<Object> exports`,
* `Local<Value> module`, and
* `Local<Context> context`

The choice to build a context-aware addon carries with it the responsibility of carefully managing global static data. Since the addon may be loaded multiple times, potentially even from different threads, any global static data stored in the addon must be properly protected, and must not contain any persistent references to JavaScript objects. The reason for this is that JavaScript objects are only valid in one context, and will likely cause a crash when accessed from the wrong context or from a different thread than the one on which they were created.

The context-aware addon can be structured to avoid global static data by performing the following steps:
* defining a class which will hold per-addon-instance data. Such a class should include a `v8::Persistent<v8::Object>` which will hold a weak reference to the addon's `exports` object. The callback associated with the weak reference will then destroy the instance of the class.
* constructing an instance of this class in the addon initializer such that the `v8::Persistent<v8::Object>` is set to the `exports` object.
* storing the instance of the class in a `v8::External`, and
* passing the `v8::External` to all methods exposed to JavaScript by passing it to the `v8::FunctionTemplate` constructor which creates the native-backed JavaScript functions. The `v8::FunctionTemplate` constructor's third parameter accepts the `v8::External`.

This will ensure that the per-addon-instance data reaches each binding that can be called from JavaScript. The per-addon-instance data must also be passed into any asynchronous callbacks the addon may create.

The following example illustrates the implementation of a context-aware addon:

```cpp
#include <node.h>

using namespace v8;

class AddonData {
 public:
  AddonData(Isolate* isolate, Local<Object> exports):
      call_count(0) {
    // Link the existence of this object instance to the existence of exports.
    exports_.Reset(isolate, exports);
    exports_.SetWeak(this, DeleteMe, WeakCallbackType::kParameter);
  }

  ~AddonData() {
    if (!exports_.IsEmpty()) {
      // Reset the reference to avoid leaking data.
      exports_.ClearWeak();
      exports_.Reset();
    }
  }

  // Per-addon data.
  int call_count;

 private:
  // Method to call when "exports" is about to be garbage-collected.
  static void DeleteMe(const WeakCallbackInfo<AddonData>& info) {
    delete info.GetParameter();
  }

  // Weak handle to the "exports" object. An instance of this class will be
  // destroyed along with the exports object to which it is weakly bound.
  v8::Persistent<v8::Object> exports_;
};

static void Method(const v8::FunctionCallbackInfo<v8::Value>& info) {
  // Retrieve the per-addon-instance data.
  AddonData* data =
      reinterpret_cast<AddonData*>(info.Data().As<External>()->Value());
  data->call_count++;
  info.GetReturnValue().Set((double)data->call_count);
}

// Initialize this addon to be context-aware.
NODE_MODULE_INIT(/* exports, module, context */) {
  Isolate* isolate = context->GetIsolate();

  // Create a new instance of AddonData for this instance of the addon.
  AddonData* data = new AddonData(isolate, exports);
  // Wrap the data in a v8::External so we can pass it to the method we expose.
  Local<External> external = External::New(isolate, data);

  // Expose the method "Method" to JavaScript, and make sure it receives the
  // per-addon-instance data we created above by passing `external` as the
  // third parameter to the FunctionTemplate constructor.
  exports->Set(context,
               String::NewFromUtf8(isolate, "method", NewStringType::kNormal)
                  .ToLocalChecked(),
               FunctionTemplate::New(isolate, Method, external)
                  ->GetFunction(context).ToLocalChecked()).FromJust();
}
```

### Building

Una volta che è stato scritto il codice sorgente, esso dev'essere compilato nel file binario `addon.node`. Per fare ciò, crea un file chiamato `binding.gyp` nel primo livello del progetto che descriva la configurazione del build del modulo usando un formato di tipo JSON. Questo file è usato da [node-gyp](https://github.com/nodejs/node-gyp) — uno strumento scritto appositamente per compilare gli Addons di Node.js.

```json
{
  "targets": [
    {
      "target_name": "addon",
      "sources": [ "hello.cc" ]
    }
  ]
}
```

Una versione dell'utility `node-gyp` è in bundle e distribuita con Node.js come parte di `npm`. Questa versione non è resa direttamente disponibile per gli sviluppatori ed è pensata solo per supportare la possibilità di utilizzare il comando `npm install` per compilare ed installare gli Addons. Gli sviluppatori che desiderano utilizzare `node-gyp` direttamente possono installarlo tramite il comando `npm install -g node-gyp`. Vedi le [istruzioni di installazione](https://github.com/nodejs/node-gyp#installation) di `node-gyp` per ulteriori informazioni, compresi i requisiti specifici della piattaforma.

Una volta creato il file `binding.gyp`, utilizzare `node-gyp configure` per generare gli appropriati build files del progetto per la piattaforma corrente. Questo genererà un `Makefile` (su piattaforme Unix) oppure un file `vcxproj` (su Windows) nella directory `build/`.

Successivamente, invoca il comando `node-gyp build` per generare il file compilato `addon.node`. Questo verrà inserito nella directory `build/Release/`.

Quando si utilizza `npm install` per installare un Addon di Node.js, npm utilizza la propria versione in bundle di `node-gyp` per eseguire lo stesso insieme di azioni, generando una versione compilata dell'Addon per la piattaforma dell'utente su richiesta.

Una volta costruito, l'Addon binario può essere utilizzato da Node.js puntando [`require()`](modules.html#modules_require) al modulo costruito `addon.node`:

```js
// hello.js
const addon = require('./build/Release/addon');

console.log(addon.hello());
// Prints: 'mondo'
```

Si prega di vedere gli esempi di seguito per ulteriori informazioni oppure <https://github.com/arturadib/node-qt> per un esempio in corso di produzione.

Poiché il percorso esatto dell'Addon binario compilato può variare a seconda di come viene compilato (ad esempio potrebbe essere in `./build/Debug/`), gli Addons possono utilizzare il [bindings](https://github.com/TooTallNate/node-bindings) package per caricare il modulo compilato.

Si noti che, mentre l'implementazione del `bindings` package è più sofisticata nel modo in cui individua i moduli Addon, sta usando essenzialmente un modello try-catch del tipo:

```js
try {
  return require('./build/Release/addon.node');
} catch (err) {
  return require('./build/Debug/addon.node');
}
```

### Collegamento alle dipendenze di Node.js

Node.js utilizza un numero di librerie collegate in modo statico come V8, libuv ed OpenSSL. Tutti gli Addons sono necessari per il collegamento a V8 ma possono anche essere collegati a qualsiasi altra dipendenza. In genere, questo è semplice come includere l'appropriato `#include <...>` le istruzioni (es. `#include <v8.h>`) e `node-gyp` individueranno automaticamente le intestazioni appropriate. Tuttavia, ci sono alcune avvertenze da tenere in considerazione:

* Quando viene eseguito `node-gyp`, esso rileverà la specifica versione di rilascio di Node.js e scaricherà il codice sorgente tarball completo oppure solo le intestazioni. Se il codice sorgente completo viene scaricato, gli Addons avranno accesso completo all'insieme di tutte le dipendenze di Node.js. Tuttavia, se vengono scaricate solo le intestazioni di Node.js, allora saranno disponibili solo i simboli esportati da Node.js.

* `node-gyp` può essere eseguito utilizzando il flag `--nodedir` che punta ad un'immagine sorgente Node.js locale. Usando questa opzione, l'Addon avrà accesso all'insieme di tutte le dipendenze.

### Caricamento degli Addons utilizzando require()

L'estensione del filename dell'Addon binario compilato è `.node` (al contrario di `.dll` oppure `.so`). La funzione [`require()`](modules.html#modules_require) viene scritta per cercare i file con l'estensione `.node` ed inizializzarli come librerie con collegamenti dinamici.

Quando si chiama [`require()`](modules.html#modules_require), solitamente l'estensione `.node` può essere omessa e Node.js continuerà a ritrovare ed inizializzare l'Addon. Un avvertimento, tuttavia, è che Node.js prima tenterà di individuare e caricare i moduli oppure i file JavaScript che capitano a condividere lo stesso nome di base. Ad esempio, se c'è un file `addon.js` nella stessa directory del file binario `addon.node`, allora [`require('addon')`](modules.html#modules_require) darà la precedenza al file `addon.js` e lo caricherà.

## Astrazioni Native per Node.Js

Ogni esempio illustrato in questo documento fa uso diretto delle API Node.js e V8 per implementare Addons. È importante capire che l'API di V8 può essere modificata radicalmente, e lo è stata, da una versione di V8 alla successiva (ancora più da una versione di Node.js alla successiva). Ad ogni modifica, potrebbe essere necessario aggiornare e ricompilare gli Addons per continuare a funzionare. La scheda di rilascio Node.js è progettata per minimizzare la frequenza e l'impatto di tali cambiamenti ma c'è una piccola parte che Node.js può attualmente eseguire, assicurando stabilità dell'API V8.

Le [Astrazioni Native per Node.js](https://github.com/nodejs/nan) (o `nan` forniscono un set di strumenti che gli sviluppatori Addon sono raccomandati ad usare per mantenere la compatibilità tra i rilasci passati e futuri di V8 e Node.js. Vedi gli [esempi](https://github.com/nodejs/nan/tree/master/examples/) `nan` per un' illustrazione di come può essere usato.

## N-API

> Stabilità: 2 - Stable

N-API è un API per costruire Addon nativi. È indipendente dal runtime JavaScript sottostante (es. V8) e viene mantenuto come parte dello stesso Node.js. Quest'API sarà stabile in Application Binary Interface (ABI) tra le versioni di Node.js. Ha lo scopo di isolare gli Addons dalle modifiche nell'engine JavaScript sottostante e consentire ai moduli compilati per una versione di essere eseguiti nelle versioni successive di Node.js senza ricompilazione. Gli Addons sono costruiti/confezionati con gli stessi metodi/strumenti evidenziati in questo documento (node-gyp, etc.). L'unica differenza è il set di API che sono usati dal codice nativo. Invece di usare le API V8 o [Astrazione Nativa per Node.js](https://github.com/nodejs/nan), sono usate le funzioni disponibili nel N-API.

Creating and maintaining an addon that benefits from the ABI stability provided by N-API carries with it certain [implementation considerations](n-api.html#n_api_implications_of_abi_stability).

Per utilizzare N-API nell'esempio precedente di "Hello world", sostituisci il contenuto di `hello.cc` con il seguente. Tutte le altre istruzioni rimangono le stesse.

```cpp
// hello.cc using N-API
#include <node_api.h>

namespace demo {

napi_value Method(napi_env env, napi_callback_info args) {
  napi_value greeting;
  napi_status status;

  status = napi_create_string_utf8(env, "world", NAPI_AUTO_LENGTH, &greeting);
  if (status != napi_ok) return nullptr;
  return greeting;
}

napi_value init(napi_env env, napi_value exports) {
  napi_status status;
  napi_value fn;

  status = napi_create_function(env, nullptr, 0, Method, nullptr, &fn);
  if (status != napi_ok) return nullptr;

  status = napi_set_named_property(env, exports, "hello", fn);
  if (status != napi_ok) return nullptr;
  return exports;
}

NAPI_MODULE(NODE_GYP_MODULE_NAME, init)

}  // namespace demo
```

Le funzioni disponibili e le informazioni su come usarle sono documentate nella sezione intitolata [Addons C/C++ - N-API](n-api.html).

## Esempi di Addons

Di seguito sono riportati alcuni esempi di Addons con lo scopo di aiutare gli sviluppatori ad iniziare. Gli esempi utilizzano le API di V8. Fare riferimento alla [V8 reference](https://v8docs.nodesource.com/) presente online per aiutarsi con le varie calls di V8, ed alla [Embedder's Guide](https://github.com/v8/v8/wiki/Embedder's%20Guide) di V8 per la spiegazione dei diversi concetti utilizzati come handles, scopes, function templates, etc.

Ciascuno di questi esempi utilizza il seguente file `binding.gyp`:

```json
{
  "targets": [
    {
      "target_name": "addon",
      "sources": [ "addon.cc" ]
    }
  ]
}
```

Nei casi in cui è presente più di un file `.cc`, basta aggiungere il filename aggiuntivo all'array `sources`:

```json
"sources": ["addon.cc", "myexample.cc"]
```

Una volta che il file `binding.gyp` è pronto, gli Addons di esempio possono essere configurati e creati usando `node-gyp`:

```console
$ node-gyp configure build
```

### Argomenti della funzione

In genere gli Addons espongono oggetti e funzioni a cui è possibile accedere da JavaScript in esecuzione all'interno di Node.js. Quando le funzioni sono invocate da JavaScript, gli argomenti di input ed il valore di return devono essere mappati sul e dal codice C/C++.

L'esempio seguente mostra come leggere gli argomenti della funzione passati da JavaScript e come restituire(return) un risultato:

```cpp
// addon.cc
#include <node.h>

namespace demo {

using v8::Exception;
using v8::FunctionCallbackInfo;
using v8::Isolate;
using v8::Local;
using v8::NewStringType;
using v8::Number;
using v8::Object;
using v8::String;
using v8::Value;

// This is the implementation of the "add" method
// Input arguments are passed using the
// const FunctionCallbackInfo<Value>& args struct
void Add(const FunctionCallbackInfo<Value>& args) {
  Isolate* isolate = args.GetIsolate();

  // Check the number of arguments passed.
  if (args.Length() < 2) {
    // Throw an Error that is passed back to JavaScript
    isolate->ThrowException(Exception::TypeError(
        String::NewFromUtf8(isolate,
                            "Wrong number of arguments",
                            NewStringType::kNormal).ToLocalChecked()));
    return;
  }

  // Check the argument types
  if (!args[0]->IsNumber() || !args[1]->IsNumber()) {
    isolate->ThrowException(Exception::TypeError(
        String::NewFromUtf8(isolate,
                            "Wrong arguments",
                            NewStringType::kNormal).ToLocalChecked()));
    return;
  }

  // Perform the operation
  double value =
      args[0].As<Number>()->Value() + args[1].As<Number>()->Value();
  Local<Number> num = Number::New(isolate, value);

  // Set the return value (using the passed in
  // FunctionCallbackInfo<Value>&)
  args.GetReturnValue().Set(num);
}

void Init(Local<Object> exports) {
  NODE_SET_METHOD(exports, "add", Add);
}

NODE_MODULE(NODE_GYP_MODULE_NAME, Init)

}  // namespace demo
```

Una volta compilato, l'Addon di esempio può essere richiesto ed utilizzato da Node.js:

```js
// test.js
const addon = require('./build/Release/addon');

console.log('This should be eight:', addon.add(3, 5));
```

### Callbacks

È pratica comune negli Addons passare le funzioni JavaScript ad una funzione C++ ed eseguirle da lì. L'esempio seguente mostra come invocare tali callbacks:

```cpp
// addon.cc
#include <node.h>

namespace demo {

using v8::Context;
using v8::Function;
using v8::FunctionCallbackInfo;
using v8::Isolate;
using v8::Local;
using v8::NewStringType;
using v8::Null;
using v8::Object;
using v8::String;
using v8::Value;

void RunCallback(const FunctionCallbackInfo<Value>& args) {
  Isolate* isolate = args.GetIsolate();
  Local<Context> context = isolate->GetCurrentContext();
  Local<Function> cb = Local<Function>::Cast(args[0]);
  const unsigned argc = 1;
  Local<Value> argv[argc] = {
      String::NewFromUtf8(isolate,
                          "hello world",
                          NewStringType::kNormal).ToLocalChecked() };
  cb->Call(context, Null(isolate), argc, argv).ToLocalChecked();
}

void Init(Local<Object> exports, Local<Object> module) {
  NODE_SET_METHOD(module, "exports", RunCallback);
}

NODE_MODULE(NODE_GYP_MODULE_NAME, Init)

}  // namespace demo
```

Nota che questo esempio usa una forma a due argomenti di `Init()` che riceve l'intero `module` object come secondo argomento. Ciò consente all'Addon di sovrascrivere completamente `exports` con una singola funzione anzichè aggiungere la funzione come una proprietà di `exports`.

Per testarlo, esegui il seguente codice JavaScript:

```js
// test.js
const addon = require('./build/Release/addon');

addon((msg) => {
  console.log(msg);
// Stampa: 'hello world'
});
```

Nota che, in questo esempio, la funzione di callback è invocata in modo sincrono.

### Object factory

Gli Addons possono creare e restituire(return) nuovi oggetti dall'interno di una funzione C++, come mostrato nel seguente esempio. Un oggetto viene creato e restituito con una proprietà `msg` che fa da eco alla stringa passata a `createObject()`:

```cpp
// addon.cc
#include <node.h>

namespace demo {

using v8::Context;
using v8::FunctionCallbackInfo;
using v8::Isolate;
using v8::Local;
using v8::NewStringType;
using v8::Object;
using v8::String;
using v8::Value;

void CreateObject(const FunctionCallbackInfo<Value>& args) {
  Isolate* isolate = args.GetIsolate();
  Local<Context> context = isolate->GetCurrentContext();

  Local<Object> obj = Object::New(isolate);
  obj->Set(context,
           String::NewFromUtf8(isolate,
                               "msg",
                               NewStringType::kNormal).ToLocalChecked(),
                               args[0]->ToString(context).ToLocalChecked())
           .FromJust();

  args.GetReturnValue().Set(obj);
}

void Init(Local<Object> exports, Local<Object> module) {
  NODE_SET_METHOD(module, "exports", CreateObject);
}

NODE_MODULE(NODE_GYP_MODULE_NAME, Init)

}  // namespace demo
```

Per testarlo in JavaScript:

```js
// test.js
const addon = require('./build/Release/addon');

const obj1 = addon('hello');
const obj2 = addon('world');
console.log(obj1.msg, obj2.msg);
// Stampa: 'hello world'
```

### Funzione factory

Un altro scenario comune è la creazione di funzioni JavaScript che racchiudono le funzioni C++ e le restituiscono a JavaScript:

```cpp
// addon.cc
#include <node.h>

namespace demo {

using v8::Context;
using v8::Function;
using v8::FunctionCallbackInfo;
using v8::FunctionTemplate;
using v8::Isolate;
using v8::Local;
using v8::NewStringType;
using v8::Object;
using v8::String;
using v8::Value;

void MyFunction(const FunctionCallbackInfo<Value>& args) {
  Isolate* isolate = args.GetIsolate();
  args.GetReturnValue().Set(String::NewFromUtf8(
      isolate, "hello world", NewStringType::kNormal).ToLocalChecked());
}

void CreateFunction(const FunctionCallbackInfo<Value>& args) {
  Isolate* isolate = args.GetIsolate();

  Local<Context> context = isolate->GetCurrentContext();
  Local<FunctionTemplate> tpl = FunctionTemplate::New(isolate, MyFunction);
  Local<Function> fn = tpl->GetFunction(context).ToLocalChecked();

  // omit this to make it anonymous
  fn->SetName(String::NewFromUtf8(
      isolate, "theFunction", NewStringType::kNormal).ToLocalChecked());

  args.GetReturnValue().Set(fn);
}

void Init(Local<Object> exports, Local<Object> module) {
  NODE_SET_METHOD(module, "exports", CreateFunction);
}

NODE_MODULE(NODE_GYP_MODULE_NAME, Init)

}  // namespace demo
```

Per testare:

```js
// test.js
const addon = require('./build/Release/addon');

const fn = addon();
console.log(fn());
// Stampa: 'hello world'
```

### Wrapping degli objects C++

È anche possibile eseguire il wrapping di oggetti/classi C++ in un modo che consenta la creazione di nuove istanze utilizzando l'operatore JavaScript `new`:

```cpp
// addon.cc
#include <node.h>
#include "myobject.h"

namespace demo {

using v8::Local;
using v8::Object;

void InitAll(Local<Object> exports) {
  MyObject::Init(exports);
}

NODE_MODULE(NODE_GYP_MODULE_NAME, InitAll)

}  // namespace demo
```

Quindi, in `myobject.h`, la classe wrapper eredita da `node::ObjectWrap`:

```cpp
// myobject.h
#ifndef MYOBJECT_H
#define MYOBJECT_H

#include <node.h>
#include <node_object_wrap.h>

namespace demo {

class MyObject : public node::ObjectWrap {
 public:
  static void Init(v8::Local<v8::Object> exports);

 private:
  explicit MyObject(double value = 0);
  ~MyObject();

  static void New(const v8::FunctionCallbackInfo<v8::Value>& args);
  static void PlusOne(const v8::FunctionCallbackInfo<v8::Value>& args);
  static v8::Persistent<v8::Function> constructor;
  double value_;
};

}  // namespace demo

#endif
```

In `myobject.cc`, implementa i vari metodi che devono essere esposti. Sotto, viene esposto il metodo `plusOne()` aggiungendolo al prototipo del constructor:

```cpp
// myobject.cc
#include "myobject.h"

namespace demo {

using v8::Context;
using v8::Function;
using v8::FunctionCallbackInfo;
using v8::FunctionTemplate;
using v8::Isolate;
using v8::Local;
using v8::NewStringType;
using v8::Number;
using v8::Object;
using v8::Persistent;
using v8::String;
using v8::Value;

Persistent<Function> MyObject::constructor;

MyObject::MyObject(double value) : value_(value) {
}

MyObject::~MyObject() {
}

void MyObject::Init(Local<Object> exports) {
  Isolate* isolate = exports->GetIsolate();

  // Prepare constructor template
  Local<FunctionTemplate> tpl = FunctionTemplate::New(isolate, New);
  tpl->SetClassName(String::NewFromUtf8(
      isolate, "MyObject", NewStringType::kNormal).ToLocalChecked());
  tpl->InstanceTemplate()->SetInternalFieldCount(1);

  // Prototype
  NODE_SET_PROTOTYPE_METHOD(tpl, "plusOne", PlusOne);

  Local<Context> context = isolate->GetCurrentContext();
  constructor.Reset(isolate, tpl->GetFunction(context).ToLocalChecked());
  exports->Set(context, String::NewFromUtf8(
      isolate, "MyObject", NewStringType::kNormal).ToLocalChecked(),
               tpl->GetFunction(context).ToLocalChecked()).FromJust();
}

void MyObject::New(const FunctionCallbackInfo<Value>& args) {
  Isolate* isolate = args.GetIsolate();
  Local<Context> context = isolate->GetCurrentContext();

  if (args.IsConstructCall()) {
    // Invoked as constructor: `new MyObject(...)`
    double value = args[0]->IsUndefined() ?
        0 : args[0]->NumberValue(context).FromMaybe(0);
    MyObject* obj = new MyObject(value);
    obj->Wrap(args.This());
    args.GetReturnValue().Set(args.This());
  } else {
    // Invoked as plain function `MyObject(...)`, turn into construct call.
    const int argc = 1;
    Local<Value> argv[argc] = { args[0] };
    Local<Function> cons = Local<Function>::New(isolate, constructor);
    Local<Object> result =
        cons->NewInstance(context, argc, argv).ToLocalChecked();
    args.GetReturnValue().Set(result);
  }
}

void MyObject::PlusOne(const FunctionCallbackInfo<Value>& args) {
  Isolate* isolate = args.GetIsolate();

  MyObject* obj = ObjectWrap::Unwrap<MyObject>(args.Holder());
  obj->value_ += 1;

  args.GetReturnValue().Set(Number::New(isolate, obj->value_));
}

}  // namespace demo
```

Per compilare questo esempio, il file `myobject.cc` deve essere aggiunto a `binding.gyp`:

```json
{
  "targets": [
    {
      "target_name": "addon",
      "sources": [
        "addon.cc",
        "myobject.cc"
      ]
    }
  ]
}
```

Testalo con:

```js
// test.js
const addon = require('./build/Release/addon');

const obj = new addon.MyObject(10);
console.log(obj.plusOne());
// Stampa: 11
console.log(obj.plusOne());
// Stampa: 12
console.log(obj.plusOne());
// Stampa: 13
```

The destructor for a wrapper object will run when the object is garbage-collected. For destructor testing, there are command-line flags that can be used to make it possible to force garbage collection. These flags are provided by the underlying V8 JavaScript engine. They are subject to change or removal at any time. They are not documented by Node.js or V8, and they should never be used outside of testing.

### Factory di wrapped objects

In alternativa, è possibile utilizzare un modello di factory per evitare la creazione esplicita di istanze di oggetti utilizzando l'operatore `new` di JavaScript:

```js
const obj = addon.createObject();
// invece di:
// const obj = new addon.Object();
```

Prima di tutto, il metodo `createObject()` è implementato in `addon.cc`:

```cpp
// addon.cc
#include <node.h>
#include "myobject.h"

namespace demo {

using v8::FunctionCallbackInfo;
using v8::Isolate;
using v8::Local;
using v8::Object;
using v8::String;
using v8::Value;

void CreateObject(const FunctionCallbackInfo<Value>& args) {
  MyObject::NewInstance(args);
}

void InitAll(Local<Object> exports, Local<Object> module) {
  MyObject::Init(exports->GetIsolate());

  NODE_SET_METHOD(module, "exports", CreateObject);
}

NODE_MODULE(NODE_GYP_MODULE_NAME, InitAll)

}  // namespace demo
```

In `myobject.h`, il metodo statico `NewInstance()` viene aggiunto per gestire la creazione di un'istanza per l'oggetto. Questo metodo sostituisce l'uso di `new` in JavaScript:

```cpp
// myobject.h
#ifndef MYOBJECT_H
#define MYOBJECT_H

#include <node.h>
#include <node_object_wrap.h>

namespace demo {

class MyObject : public node::ObjectWrap {
 public:
  static void Init(v8::Isolate* isolate);
  static void NewInstance(const v8::FunctionCallbackInfo<v8::Value>& args);

 private:
  explicit MyObject(double value = 0);
  ~MyObject();

  static void New(const v8::FunctionCallbackInfo<v8::Value>& args);
  static void PlusOne(const v8::FunctionCallbackInfo<v8::Value>& args);
  static v8::Persistent<v8::Function> constructor;
  double value_;
};

}  // namespace demo

#endif
```

L'implementazione in `myobject.cc` è simile all'esempio precedente:

```cpp
// myobject.cc
#include <node.h>
#include "myobject.h"

namespace demo {

using v8::Context;
using v8::Function;
using v8::FunctionCallbackInfo;
using v8::FunctionTemplate;
using v8::Isolate;
using v8::Local;
using v8::NewStringType;
using v8::Number;
using v8::Object;
using v8::Persistent;
using v8::String;
using v8::Value;

Persistent<Function> MyObject::constructor;

MyObject::MyObject(double value) : value_(value) {
}

MyObject::~MyObject() {
}

void MyObject::Init(Isolate* isolate) {
  // Prepare constructor template
  Local<FunctionTemplate> tpl = FunctionTemplate::New(isolate, New);
  tpl->SetClassName(String::NewFromUtf8(
      isolate, "MyObject", NewStringType::kNormal).ToLocalChecked());
  tpl->InstanceTemplate()->SetInternalFieldCount(1);

  // Prototype
  NODE_SET_PROTOTYPE_METHOD(tpl, "plusOne", PlusOne);

  Local<Context> context = isolate->GetCurrentContext();
  constructor.Reset(isolate, tpl->GetFunction(context).ToLocalChecked());
}

void MyObject::New(const FunctionCallbackInfo<Value>& args) {
  Isolate* isolate = args.GetIsolate();
  Local<Context> context = isolate->GetCurrentContext();

  if (args.IsConstructCall()) {
    // Invoked as constructor: `new MyObject(...)`
    double value = args[0]->IsUndefined() ?
        0 : args[0]->NumberValue(context).FromMaybe(0);
    MyObject* obj = new MyObject(value);
    obj->Wrap(args.This());
    args.GetReturnValue().Set(args.This());
  } else {
    // Invoked as plain function `MyObject(...)`, turn into construct call.
    const int argc = 1;
    Local<Value> argv[argc] = { args[0] };
    Local<Function> cons = Local<Function>::New(isolate, constructor);
    Local<Object> instance =
        cons->NewInstance(context, argc, argv).ToLocalChecked();
    args.GetReturnValue().Set(instance);
  }
}

void MyObject::NewInstance(const FunctionCallbackInfo<Value>& args) {
  Isolate* isolate = args.GetIsolate();

  const unsigned argc = 1;
  Local<Value> argv[argc] = { args[0] };
  Local<Function> cons = Local<Function>::New(isolate, constructor);
  Local<Context> context = isolate->GetCurrentContext();
  Local<Object> instance =
      cons->NewInstance(context, argc, argv).ToLocalChecked();

  args.GetReturnValue().Set(instance);
}

void MyObject::PlusOne(const FunctionCallbackInfo<Value>& args) {
  Isolate* isolate = args.GetIsolate();

  MyObject* obj = ObjectWrap::Unwrap<MyObject>(args.Holder());
  obj->value_ += 1;

  args.GetReturnValue().Set(Number::New(isolate, obj->value_));
}

}  // namespace demo
```

Ancora una volta, per compilare questo esempio, il file `myobject.cc` deve essere aggiunto a `binding.gyp`:

```json
{
  "targets": [
    {
      "target_name": "addon",
      "sources": [
        "addon.cc",
        "myobject.cc"
      ]
    }
  ]
}
```

Testalo con:

```js
// test.js
const createObject = require('./build/Release/addon');

const obj = createObject(10);
console.log(obj.plusOne());
// Stampa: 11
console.log(obj.plusOne());
// Stampa: 12
console.log(obj.plusOne());
// Stampa: 13

const obj2 = createObject(20);
console.log(obj2.plusOne());
// Stampa: 21
console.log(obj2.plusOne());
// Stampa: 22
console.log(obj2.plusOne());
// Stampa: 23
```

### Riavvolgere gli wrapped objects

Oltre ad eseguire il wrapping e restituire objects C++, è possibile riavvolgere gli wrapped objects tramite l'unwrapping con la funzione di supporto di Node.js `node::ObjectWrap::Unwrap`. I seguenti esempi mostrano una funzione `add()` che può prendere due oggetti `MyObject` come argomenti di input:

```cpp
// addon.cc
#include <node.h>
#include <node_object_wrap.h>
#include "myobject.h"

namespace demo {

using v8::Context;
using v8::FunctionCallbackInfo;
using v8::Isolate;
using v8::Local;
using v8::Number;
using v8::Object;
using v8::String;
using v8::Value;

void CreateObject(const FunctionCallbackInfo<Value>& args) {
  MyObject::NewInstance(args);
}

void Add(const FunctionCallbackInfo<Value>& args) {
  Isolate* isolate = args.GetIsolate();
  Local<Context> context = isolate->GetCurrentContext();

  MyObject* obj1 = node::ObjectWrap::Unwrap<MyObject>(
      args[0]->ToObject(context).ToLocalChecked());
  MyObject* obj2 = node::ObjectWrap::Unwrap<MyObject>(
      args[1]->ToObject(context).ToLocalChecked());

  double sum = obj1->value() + obj2->value();
  args.GetReturnValue().Set(Number::New(isolate, sum));
}

void InitAll(Local<Object> exports) {
  MyObject::Init(exports->GetIsolate());

  NODE_SET_METHOD(exports, "createObject", CreateObject);
  NODE_SET_METHOD(exports, "add", Add);
}

NODE_MODULE(NODE_GYP_MODULE_NAME, InitAll)

}  // namespace demo
```

In `myobject.h`, viene aggiunto un nuovo metodo pubblico per consentire l'accesso ai valori privati dopo aver eseguito l'unwrapping dell'oggetto.

```cpp
// myobject.h
#ifndef MYOBJECT_H
#define MYOBJECT_H

#include <node.h>
#include <node_object_wrap.h>

namespace demo {

class MyObject : public node::ObjectWrap {
 public:
  static void Init(v8::Isolate* isolate);
  static void NewInstance(const v8::FunctionCallbackInfo<v8::Value>& args);
  inline double value() const { return value_; }

 private:
  explicit MyObject(double value = 0);
  ~MyObject();

  static void New(const v8::FunctionCallbackInfo<v8::Value>& args);
  static v8::Persistent<v8::Function> constructor;
  double value_;
};

}  // namespace demo

#endif
```

L'implementazione di `myobject.cc` è simile a prima:

```cpp
// myobject.cc
#include <node.h>
#include "myobject.h"

namespace demo {

using v8::Context;
using v8::Function;
using v8::FunctionCallbackInfo;
using v8::FunctionTemplate;
using v8::Isolate;
using v8::Local;
using v8::NewStringType;
using v8::Object;
using v8::Persistent;
using v8::String;
using v8::Value;

Persistent<Function> MyObject::constructor;

MyObject::MyObject(double value) : value_(value) {
}

MyObject::~MyObject() {
}

void MyObject::Init(Isolate* isolate) {
  // Prepare constructor template
  Local<FunctionTemplate> tpl = FunctionTemplate::New(isolate, New);
  tpl->SetClassName(String::NewFromUtf8(
      isolate, "MyObject", NewStringType::kNormal).ToLocalChecked());
  tpl->InstanceTemplate()->SetInternalFieldCount(1);

  Local<Context> context = isolate->GetCurrentContext();
  constructor.Reset(isolate, tpl->GetFunction(context).ToLocalChecked());
}

void MyObject::New(const FunctionCallbackInfo<Value>& args) {
  Isolate* isolate = args.GetIsolate();
  Local<Context> context = isolate->GetCurrentContext();

  if (args.IsConstructCall()) {
    // Invoked as constructor: `new MyObject(...)`
    double value = args[0]->IsUndefined() ?
        0 : args[0]->NumberValue(context).FromMaybe(0);
    MyObject* obj = new MyObject(value);
    obj->Wrap(args.This());
    args.GetReturnValue().Set(args.This());
  } else {
    // Invoked as plain function `MyObject(...)`, turn into construct call.
    const int argc = 1;
    Local<Value> argv[argc] = { args[0] };
    Local<Function> cons = Local<Function>::New(isolate, constructor);
    Local<Object> instance =
        cons->NewInstance(context, argc, argv).ToLocalChecked();
    args.GetReturnValue().Set(instance);
  }
}

void MyObject::NewInstance(const FunctionCallbackInfo<Value>& args) {
  Isolate* isolate = args.GetIsolate();

  const unsigned argc = 1;
  Local<Value> argv[argc] = { args[0] };
  Local<Function> cons = Local<Function>::New(isolate, constructor);
  Local<Context> context = isolate->GetCurrentContext();
  Local<Object> instance =
      cons->NewInstance(context, argc, argv).ToLocalChecked();

  args.GetReturnValue().Set(instance);
}

}  // namespace demo
```

Testalo con:

```js
// test.js
const addon = require('./build/Release/addon');

const obj1 = addon.createObject(10);
const obj2 = addon.createObject(20);
const result = addon.add(obj1, obj2);

console.log(result);
// Stampa: 30
```

### AtExit hooks

Un hook `AtExit` è una funzione che viene invocata dopo la fine del ciclo di eventi di Node.js ma prima che la JavaScript VM sia terminata e prima che Node.js si arresti. Gli hooks `AtExit` vengono registrati utilizzando l'API `node::AtExit`.

#### void AtExit(callback, args)

* `callback` <span class="type">&lt;void (\*)(void\*)&gt;</span> A pointer to the function to call at exit.
* `args` <span class="type">&lt;void\*&gt;</span> Un puntatore per passare ad un "callback at exit".

Registra gli "exit hooks" che vengono eseguiti dopo che il ciclo di eventi è terminato ma prima che la VM venga distrutta.

`AtExit` accetta due parametri: un puntatore ad una funzione di callback da eseguire all'uscita (at exit), ed un puntatore a dati contestuali untyped da passare a tale callback.

I callback vengono eseguiti nell'ordine last-in first-out (ultimo ad entrare, primo ad uscire).

Il seguente `addon.cc` implementa `AtExit`:

```cpp
// addon.cc
#include <assert.h>
#include <stdlib.h>
#include <node.h>

namespace demo {

using node::AtExit;
using v8::HandleScope;
using v8::Isolate;
using v8::Local;
using v8::Object;

static char cookie[] = "yum yum";
static int at_exit_cb1_called = 0;
static int at_exit_cb2_called = 0;

static void at_exit_cb1(void* arg) {
  Isolate* isolate = static_cast<Isolate*>(arg);
  HandleScope scope(isolate);
  Local<Object> obj = Object::New(isolate);
  assert(!obj.IsEmpty());  // Afferma che la VM è ancora funzionante
  assert(obj->IsObject());
  at_exit_cb1_called++;
}

static void at_exit_cb2(void* arg) {
  assert(arg == static_cast<void*>(cookie));
  at_exit_cb2_called++;
}

static void sanity_check(void*) {
  assert(at_exit_cb1_called == 1);
  assert(at_exit_cb2_called == 2);
}

void init(Local<Object> exports) {
  AtExit(at_exit_cb2, cookie);
  AtExit(at_exit_cb2, cookie);
  AtExit(at_exit_cb1, exports->GetIsolate());
  AtExit(sanity_check);
}

NODE_MODULE(NODE_GYP_MODULE_NAME, init)

}  // namespace demo
```

Testa in JavaScript eseguendo:

```js
// test.js
require('./build/Release/addon');
```
