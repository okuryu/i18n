# C++ Addons

<!--introduced_in=v0.10.0-->
<!-- type=misc -->

Node.js Addons zijn dynamisch gekoppelde gedeelde objecten, geschreven in C++ die in Node.js kunnen worden geladen met behulp van de [`require()`](modules.html#modules_require)functie, en kunnen worden gebruikt alsof zij een gewone Node.js module zijn. Ze worden voornamelijk gebruikt als interface tussen JavaScript uitgevoerd in Node.js en C/C++ bibliotheken.

Op dit moment is de methode voor het implementeren van Addons nogal ingewikkeld, waarbij kennis van de verschillende componenten en API's noodzakelijk is:

 - V8: de C++ bibliotheek die Node.js momenteel gebruikt ten behoeve van de uitvoering van JavaScript. V8 biedt het mechanisme voor het creëren van objecten, aanroepfuncties, enz. V8 API wordt meestal gedocumenteerd in het `v8.h`headerbestand (`deps/v8/include/v8.h` in de Node.js source tree), die ook beschikbaar is [online](https://v8docs.nodesource.com/).

 - [libuv](https://github.com/libuv/libuv): De C bibliotheek die de Node.js gebeurtenissenlus, zijn werk-items, en alle asynchrone procesvoering van het platform implementeert. Het fungeert ook als een platformoverschrijdende abstractie bibliotheek, wat gemakkelijke, POSIX-achtige toegang geeft tot alle belangrijke besturingssystemen naar populaire systeemtaken, zoals de interactie met het bestandssysteem, sockets, timers, en systeemgebeurtenissen. libuv biedt ook een pthreads-achtige threading abstractie, die kan worden ingezet om meer kracht te geven aan meer geavanceerde asynchrone addons die verder moeten gaan dan de standaard gebeurtenis-iteratie. Addon auteurs worden aangemoedigd om na te denken over hoe ze kunnen voorkomen dat een gebeurtenissenlus met I/O en andere tijdsintensieve taken wordt geblokkeerd, door het werk te off-loaden via libuv naar niet-blokkerende systeem operaties, werk-threads of een aangepast gebruik van libuv's threads.

 - Interne Node.js bibliotheken. Node.js exporteert zelf een aantal C++ APIs die Addons kunnen gebruiken &mdash; de meest belangrijke daarvan is de `node::ObjectWrap` klasse.

 - Node.js bevat een aantal andere statisch gekoppelde bibliotheken, zoals OpenSSL. Deze andere bibliotheken bevinden zich in de `deps/` map in de Node.js source tree. Alleen de libuv, OpenSSL, V8 en zlib symbolen zijn doelbewust wederuitgevoerd door Node.js en kunnen in verschillende mate door Addons worden gebruikt. Zie [Link naar eigen afhankelijkheden van Node.js](#addons_linking_to_node_js_own_dependencies) voor meer informatie.

Alle van de volgende voorbeelden zijn beschikbaar om te [downloaden](https://github.com/nodejs/node-addon-examples) en kunnen worden gebruikt als uitgangspunt voor een Addon.

## Hallo wereld

Dit "Hallo wereld" voorbeeld is een simpele Addon, geschreven in C++, en is gelijkwaardig aan de volgende JavaScript-code:

```js
module.exports.hallo = () => 'wereld';
```

Maak eerst het bestand `hallo.cc`:

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

Hou er rekening mee dat alle Node.js Addons een initialisatie functie moeten exporteren, volgens het patroon:

```cpp
void Initialize(Local<Object> exports);
NODE_MODULE(NODE_GYP_MODULE_NAME, Initialize)
```

Er is geen puntkomma na `NODE_MODULE` omdat het geen functie is (zie `node.h`).

De `module_name` moet overeenstemmen met de bestandsnaam van de laatste binary (exclusief het `.node` achtervoegsel).

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

### Bouwen

Zodra de broncode is geschreven, moet het in het binaire `addon.node` bestand worden gecompileerd. Om dit te doen, maak een bestand genaamd `binding.gyp` in het top-level van het project met een beschrijving van de bouwconfiguratie van de module met behulp van een JSON-achtig format. Dit bestand wordt gebruikt door [node-gyp](https://github.com/nodejs/node-gyp) — een tool die specifiek is geschreven om Node.js Addons te compileren.

```json
{
  "targets": [
    {
      "target_name": "addon",
      "sources": [ "hallo.cc" ]
    }
  ]
}
```

Een versie van het `node-gyp` hulpprogramma is gebundeld en gedistribueerd met Node.js als onderdeel van `npm`. Deze versie is niet direct beschikbaar gesteld om door ontwikkelaars gebruikt te worden en dient alleen ter ondersteuning van de mogelijkheid om de `npm install` opdracht te compileren en het installeren van Addons. Ontwikkelaars die rechtstreeks gebruik willen maken van `node-gyp`, kunnen dit installeren met behulp van de opdracht `npm install -g node-gyp`. Zie de `node-gyp` [installatie instructies](https://github.com/nodejs/node-gyp#installation) voor meer informatie, inclusief platform-specifieke eisen.

Wanneer het `binding.gyp` bestand eenmaal is gemaakt, gebruik dan `node-gyp configure` om de passende project bouwbestanden voor het huidige platform te genereren. Dit genereert ofwel een `Makefile` (op Unix platforms) of een `vcxproj` bestand (op Windows) in de `build/` map.

Vervolgens, roep de `node-gyp build` opdracht aan om het gecompileerde `addon.node` bestand te genereren. Dit wordt dan in de `build/Release/` map gezet.

Bij gebruik van `npm install` voor het installeren van een Node.js Addon, maakt npm gebruik van de eigen gebundelde versie van `node-gyp` om dezelfde acties uit te voeren, het genereren van een gecompileerde versie van de Addon, voor het gebruikersplatform op aanvraag.

Eenmaal gebouwd, kan de binary Addon worden gebruikt binnen Node.js door [`require()`](modules.html#modules_require) te wijzen naar de gebouwde `addon.node` module:

```js
// hallo.js
const addon = require('./build/Release/addon');

console.log(addon.hallo());
// Prints: 'wereld'
```

Zie alsjeblieft hieronder de voorbeelden voor meer informatie of <https://github.com/arturadib/node-qt> voor een voorbeeld in de productie.

Omdat het exacte pad naar de gecompileerde Addon binary kan variëren, afhankelijk van hoe het is gecompileerd (d.w.z. soms is het in `./build/Debug/`), kunnen Addons het [bindings](https://github.com/TooTallNate/node-bindings) pakket gebruiken om de gecompileerde module te laden.

Merk hierbij op dat terwijl de `bindings` pakket uitvoering geavanceerder is in hoe het Addon modules zoekt, het is in feite met behulp van een try-catch patroon te vergelijken met:

```js
try {
  return require('./build/Release/addon.node');
} catch (err) {
  return require('./build/Debug/addon.node');
}
```

### Koppelen aan de eigen afhankelijkheden van Node.js

Node.js gebruikt een aantal statisch gekoppelde bibliotheken zoals V8, libuv en OpenSSL. Alle Addons zijn verplicht te linken naar V8 en mogen daarnaast ook naar de andere afhankelijkheden linken. Gewoonlijk is dit zo simpel als het insluiten van de passende `#include <...>` verklaring ( bijv. `#include <v8.h>`) en `node-gyp` zal automatisch de passende titels vinden. Er zijn echter een paar uitzonderingen om zich bewust van te zijn:

* Wanneer `node-gyp` uitgevoerd wordt, zal het de specifieke gepubliceerde versie van Node.js detecteren en ofwel de volledige bron tarball downloaden of alleen de titels. Wanneer de volledige bron is gedownload, zullen Addons volledige toegang hebben tot de volledige set van Node.js afhankelijkheden. Echter, wanneer alleen de Node.js titels zijn gedownload, dan zijn alleen de symbolen die zijn geëxporteerd door Node.js beschikbaar.

* `node-gyp` kan worden uitgevoerd met behulp van de `--nodedir` wijzend naar de locale Node.js bronafbeelding. Wanneer deze optie gebruikt wordt, heeft de Addon volledige toegang tot de volledige set van afhankelijkheden.

### Addons laden met behulp van require()

De bestandsnaam-extensie van het gecompileerde Addon binair is `.node` (in tegenstelling tot `.dll` or `.so`). De [`require()`](modules.html#modules_require) functie is geschreven om te zoeken naar bestanden met de `.node` bestands-extensie en initialiseert deze als dynamisch gekoppelde bibliotheken.

Bij het aanroepen van [`require()`](modules.html#modules_require), kan de `.node` extensie meestal worden weggelaten en Node.js zal de Addon nog steeds vinden en initialiseren. Één uitzondering is echter, dat Node.js eerst zal proberen modules of JavaScript bestanden te vinden en laden die wellicht dezelfde basisnaam delen. Bijvoorbeeld, als een bestand `addon.js` in dezelfde map zit als de binair `addon.node`, dan zal [`require('addon')`](modules.html#modules_require) voorkeur geven aan het bestand `addon.js`, en die in plaats daarvan laden.

## Oorspronkelijke abstracties voor Node.js

Elk van de voorbeelden, die zijn weergeven in dit document, maken direct gebruik van de Node.js en V8 API's voor de uitvoering van Addons. Het is belangrijk dat men begrijpt dat de V8-API dramatisch kan veranderen, en dit is ook al het geval geweest, van één V8 uitgave tot de volgende (en van een belangrijke Node.js uitgave tot de volgende). Bij elke wijziging zou het kunnen dat de Addons moeten worden bijgewerkt, en opnieuw gecompileerd om te blijven functioneren. Het uitgave-schema van Node.js is ontworpen om de frequentie en impact van dergelijke veranderingen te minimaliseren, maar Node.js kan momenteel weinig doen om voor stabiliteit van de V8-API's te zorgen.

De [ Oorspronkelijke Abstracties voor Node.js](https://github.com/nodejs/nan) (of `nan`) verschaffen een set hulpmiddelen die aanbevolen zijn om te worden gebruikt door Addon ontwikkelaars, om overeenstemming tussen oude en toekomstige uitgaven van V8 en Node.js te bewaren. Zie de `nan`[voorbeelden](https://github.com/nodejs/nan/tree/master/examples/) voor een voorbeeld over hoe dit kan worden gebruikt.

## N-API

> Stabiliteit: 2 - stabiel

N-API is een API voor het bouwen van oorspronkelijke Addons. Het is onafhankelijk van de onderliggende JavaScript runtime (bijv. V8) en wordt onderhouden als een deel van Node.js zelf. This API will be Application Binary Interface (ABI) stable across versions of Node.js. Het is bedoeld om Addons te isoleren van veranderingen in de onderliggende JavaScript motor, én het mogelijk te maken voor modules die samengesteld zijn om op één versie te draaien, dat zij dat ook op nieuwere versies van Node.js kunnen zonder dat zij opnieuw gecompileerd moeten worden. Addons zijn gebouwd/verpakt met dezelfde aanpak/hulpmiddelen die beschreven worden in dit document (node-gyp, etc.). Het enige verschil is de set API's die zijn gebruikt bij de oorspronkelijke code. In plaats van de V8 of [Oorspronkelijke Abstracties voor Node.js](https://github.com/nodejs/nan) API's, worden de beschikbare functies in de N-API gebruikt.

Creating and maintaining an addon that benefits from the ABI stability provided by N-API carries with it certain [implementation considerations](n-api.html#n_api_implications_of_abi_stability).

Om de N-API in het bovenstaande "Hallo wereld" voorbeeld te gebruiken, vervang de inhoud van `hallo.cc` met het volgende. Alle andere instructies blijven hetzelfde.

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

De beschikbare functies, en hoe ze gebruikt moeten worden, staan beschreven in de sectie met de titel [C/C++ Addons - N-API](n-api.html).

## Addon voorbeelden

Hier volgen enkele voorbeelden van Addons die bedoeld zijn om ontwikkelaars te helpen aan de slag te gaan. De voorbeelden maken gebruik van de V8-API's. Raadpleeg de online [V8 verwijzing](https://v8docs.nodesource.com/) voor hulp bij de verschillende V8 oproepen, en V8 [Embedder's Guide](https://github.com/v8/v8/wiki/Embedder's%20Guide) voor uitleg over de verschillende concepten zoals handvatten, bereiken, functie patronen, etc.

Elk van deze voorbeelden met gebruik van het volgende `binding.gyp` bestand:

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

In gevallen waar er meer dan één `.cc` bestand is, voeg simpelweg de extra bestandsnaam aan de reeks van `bronnen` toe:

```json
"bronnen": ["addon.cc", "myexample.cc"]
```

Zodra het `binding.gyp` klaar is, kunnen de voorbeeld-Addons worden geconfigureerd en gebouwd met behulp van `node-gyp`:

```console
$ node-gyp configure build
```

### Functie argumenten

Addons zullen meestal objecten en functies blootleggen die toegankelijk zijn vanuit JavaScript uitgevoerd binnen Node.js. Wanneer functies worden aangeroepen vanuit JavaScript, moeten de invoer-argumenten en de geretourneerde waarde worden toegewezen van, én naar de C/C++ code.

Het volgende voorbeeld laat zien hoe een functie-argument wat is doorgegeven van JavaScript gelezen moet worden, en hoe het resultaat geretourneerd moet worden:

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

Eenmaal gecompileerd, kan het voorbeeld Addon vereist zijn, en gebruikt worden vanuit Node.js:

```js
// test.js
const addon = require('./build/Release/addon');

console.log('This should be eight:', addon.add(3, 5));
```

### Callbacks

Het is gebruikelijk binnen Addons om JavaScript functies door te geven naar een C++ functie en deze daar vanuit te executeren. Het volgende voorbeeld laat zien hoe dergelijke callbacks worden aangeroepen:

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

Merk op dat dit voorbeeld een twee-argument vorm gebruikt van `Init()` dat het volledige `module` object als tweede argument gebruikt. Dit maakt het de Addon mogelijk om `exports` geheel te overschrijven met één enkele functie, in plaats van het toevoegen van de functie als bezit van `exports`.

Om dit te testen, draai het volgende JavaScript:

```js
// test.js
const addon = require('./build/Release/addon');

addon((msg) => {
  console.log(msg);
// Prints: 'hallo wereld'
});
```

Merk hierbij op dat, in dit voorbeeld, tegelijkertijd de callback functie wordt aangeroepen.

### Object fabriek

Zoals je kunt zien in het volgende voorbeeld, kunnen Addons nieuwe objecten creëren en retourneren vanuit een C++ functie. Een object wordt gecreëerd en geretourneerd met een eigenschap `msg` die de tekenreeks doorgegeven aan `createObject()`: herhaalt.

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

Om dit te testen in JavaScript:

```js
// test.js
const addon = require('./build/Release/addon');

const obj1 = addon('hallo');
const obj2 = addon('wereld');
console.log(obj1.msg, obj2.msg);
// Prints: 'hallo wereld'
```

### Functie fabriek

Een ander veelvoorkomend scenario is het creëren van JavaScript functies die C++ functies inpakken en deze retourneren naar JavaScript:

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

Om te testen:

```js
//test.js
const addon = require('./build/Release/addon');

const fn = addon();
console.log(fn());
// Prints: 'hallo wereld'
```

### C++ objecten inpakken

Het is ook mogelijk om C++ objecten/klassen in te pakken, op een zodanige manier dat het toelaat nieuwe exemplaren te creëren met behulp van JavaScript `new` operator:

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

Vervolgens erft de inpak-klasse van `node::ObjectWrap`: in `myobject.h`

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

Pas de verschillende methoden die onthult moeten worden toe in `myobject.cc`. Hieronder wordt de methode `plusOne()` onthult door ze toe te voegen aan het prototype van de ontwikkelaar:

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

Om dit voorbeeld te bouwen, moet het `myobject.cc` bestand toegevoegd worden aan de `binding.gyp`:

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

Test dit met:

```js
// test.js
const addon = require('./build/Release/addon');

const obj = new addon.MyObject(10);
console.log(obj.plusOne());
// Prints: 11
console.log(obj.plusOne());
// Prints: 12
console.log(obj.plusOne());
// Prints: 13
```

The destructor for a wrapper object will run when the object is garbage-collected. For destructor testing, there are command-line flags that can be used to make it possible to force garbage collection. These flags are provided by the underlying V8 JavaScript engine. They are subject to change or removal at any time. They are not documented by Node.js or V8, and they should never be used outside of testing.

### Fabriek van ingepakte objecten

Als alternatief, is het mogelijk om een fabriekspatroon te gebruiken om het expliciet maken van objectexemplaren te voorkomen met behulp van de JavaScript `new` operator:

```js
const obj = addon.createObject();
// instead of:
// const obj = new addon.Object();
```

Eerst wordt de `createObject()` methode geïmplementeerd in `addon.cc`:

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

In `myobject.h` wordt de statische methode `NewInstance()` toegevoegd om mogelijk te maken het object te vinden. Deze methode wordt gebruikt in plaats van `new` in JavaScript:

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

De uitvoering in `myobject.cc` is te gebruiken met het vorige voorbeeld:

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

Nogmaals, om dit voorbeeld te bouwen, moet het `myobject.cc` bestand toegevoegd worden aan de `binding.gyp`:

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

Test dit met:

```js
// test.js
const createObject = require('./build/Release/addon');

const obj = createObject(10);
console.log(obj.plusOne());
// Prints: 11
console.log(obj.plusOne());
// Prints: 12
console.log(obj.plusOne());
// Prints: 13

const obj2 = createObject(20);
console.log(obj2.plusOne());
// Prints: 21
console.log(obj2.plusOne());
// Prints: 22
console.log(obj2.plusOne());
// Prints: 23
```

### Ingepakte objecten doorgeven

Naast het verpakken en retourneren van C++ objecten, is het mogelijk om ingepakte objecten door te geven door ze uit te pakken met behulp van de Node.js hulp functie. `node::ObjectWrap::Unwrap`. De volgende voorbeelden laten de `add()` functie zien, die twee `MyObject` objecten als invoerargumenten kan nemen:

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

In `myobject.h` is een nieuwe publieke methode toegevoegd die toegang geeft tot de privé-waarden, na het uitpakken van het object.

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
```

De uitvoering van `myobject.cc` is gelijkwaardig aan het vorige voorbeeld:

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

Test dit met:

```js
// test.js
const addon = require('./build/Release/addon');

const obj1 = addon.createObject(10);
const obj2 = addon.createObject(20);
const result = addon.add(obj1, obj2);

console.log(result);
// Prints: 30
```

### AtExit haken

Een `AtExit` haak is een functie die is opgeroepen nadat de Node.js gebeurtenis-lus is afgelopen maar vóórdat JavaScript VM is beëindigd en Node.js afsluit. `AtExit` haken worden geregistreerd met behulp van de `node::AtExit` API.

#### void AtExit(callback, args)

* `callback` <span class="type">&lt;void (\*)(void\*)&gt;</span> A pointer to the function to call at exit.
* `args` <span class="type">&lt;void\*&gt;</span> Een pointer om aan de callback door te geven bij het afsluiten.

Registreert exit haken die draaien nadat de gebeurtenis-lus is beëindigd, maar vóórdat de VM wordt afgesloten.

`AtExit` neemt twee parameters: een pointer naar een callback functie om te draaien bij het afsluiten, en een pointer naar ongetypte context data wat doorgegeven moet worden naar die callback.

Callbacks worden gedraaid in 'laatste in, eerste uit' volgorde.

De volgende `addon.cc` implementeert `AtExit`:

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
  assert(!obj.IsEmpty());  // assert VM is still alive
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

Test in JavaScript door het volgende te laten draaien:

```js
// test.js
require('./build/Release/addon');
```
