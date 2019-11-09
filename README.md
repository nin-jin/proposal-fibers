# Fibers

- [Motivation](#motivation)
- [API Changes](#api-changes)
- [Usage examples](#usage-examples)
- [Limitations](#limitations)
- [Related Implementations](#related-implementations)
- [Other languages](#other-languages)

## Motivation

Pausable stackfull coroutines (fibers) give ability to write simple but responsive and cuncurrently code.

It can be used for:
1. Ability to call async code from sync code.
2. Automatic pause long job every 16 ms to ensure 60fps.
3. Concurent execution of different long jobs (not serial).
4. Abort not completed long job (with subjobs) as reaction on event.
5. Abstract of asynchrony. Synchronous code is simplier and can be better optimized by JIT.

## API

```typescript
// Fiber - lightweight thread, that has separate call stack.
// Multiple fibers concurrently executes in one system thread.
declare class Fiber< Result > implements PromiseLike< Result > {

	// Fiber that are executing now.
	static current : Fiber< any > | null

	// Freezes current fiber until promise will be resolved.
	// After, resumes fiber and returns result or rethrows an exception if promise is rejected.
	// Throws NotInFiberError if called outside any fiber.
	static wait< Result >( promise : PromiseLike< Result > ) : Result

	// Executes function in separate fiber
	static run( task : ()=> Result ) : Fiber< Result >

	// Alias for `Fiber.wait( fiber.run() )`
	wait() : Result

	// Abort execution.
	// Do nothing if is already completed.
	abort : ()=> void

}
```

## Special syntax

**`fiber`** keyword can be used to define function that always start new fiber on every call:

```
fiber function test() {
    [ 1 , 2 , 3 ].map( i => { // sync map
    	console.log( Fiber.wait( fetch( `/ping${i}` ) ) )
    } )
}
```

```
const test = fiber () => {
    [ 1 , 2 , 3 ].map( i => { // sync map
    	console.log( Fiber.wait( fetch( `/ping${i}` ) ) )
    } )
}
```

## Usage examples

- [Quantizing](#quantizing)
- [60 FPS rendering](#60-fps-rendering)
- [Optional Fetch](#optional-fetch)
- [JSON processing](#json-processing)

### Quantizing

Function that returns control to main loop every 8 ms:

```typescript
let deadline = Date.now() + 8

function quant() {

	// Do nothing if fibers isn't supported 
	if( typeof Fiber !== 'function' ) return

	// Do nothing if called outside any fiber
	if( !Fiber.current ) return

	// Do nothing until deadline
	if( Date.now() < deadline ) return

	// Stop current fiber until next frame
	Fiber.wait( new Promise( requestAnimationFrame ) )

	// already in next animation frame
	dealine = Date.now() + 8

}
```

### 60 FPS rendering

Very huge rendering (over 9000ms):

```typescript
// Render large tree.
function render_tree( count ) {

	// [ 0 , 1 , 2 , ... ]
	const numbers = [ ... Array( count ) ].map( ( _ , i ) => i )

	// [ <iframe/> , <iframe/> , ... ]
	const leafs = render_branch( document.body , numbers )

	console.log( leafs )
}

// Render branch.
function render_branch( parent , numbers ) {

	// Render element for branch.
	const el = document.createElement( 'div' )
	parent.appendChild( el )
	
	// Render few leafs and return its.
	if( numbers.length <= 5 ) return numbers.map( render_leaf.bind( el ) )

	// Split numbers to two buckets.
	const center = Math.floor( numbers.length / 2 )
	const left = numbers.slice( 0 , center )
	const right = numbers.slice( center )
	
	// Render sub branches and join returned leafs
	return [ ... render_branch( el , left ) , ... render_branch( el , right ) ]
	
}

function render_leaf( parent ) {
	
	// Ability to free main thread every 8 ms
	quant()
	
	// Do hard work (iframes are very expansive)
	const el = document.createElement( 'iframe' )
	parent.appendChild( el )
	
	return el
}

```

Start rendering in separate fiber:

```typescript
let rendering = Fiber.run( ()=> render_tree( 1000 ) )
```

Abort old rendering and start new on event:

```typescript
// On any external event
window.onmessage( count => {
  
	// Stop revious rendering
	rendering.abort()
	
	// Start new rendering
	rendering = Fiber.run( ()=> render_tree( count ) )
  
} )
```

### Optional Fetch

Simple `fetch` wrapper:

```typescript
function get_json( url : string ) : any {

	const controller = new AbortController();
	const signal = controller.signal;

	// Wait for server response
	const response = Fiber.wait( fetch( url , { signal } ) )

	// Wait for json parse
	const json = Fiber.wait( response.json() )

	return json	
}
```

Optional asynchrony in other code:

```typescript
let cache : { beta : boolean }

function get_config() : typeof cache {

	// Fill cache if not exists 
	if( !cache ) cache = get_json( 'example.org' ).flags

	return cache
}

Fiber.run( ()=> {

	// Will suspend while htt requesting
	console.log( get_config() )

	// Instant returns data from cache
	console.log( get_config() )

} )
```

### JSON processing

Quantized json parsing that don't block event loop more than ~8 ms independent on string size:

```typescript
json_parse( str ) : any {
	return JSON.parse( str , ( key , value )=> ( quant() , value ) )
}
```

Usage:

```typescript
import { promisify } from 'util'
import { readFile } from 'fs'
const readFileAsync = promisify( readFile )

// Some synchronous task
function log_config() {

	// Load large JSON
	const buffer = Fiber.wait( fs.readFileASync( 'config.json' ) )

	// Parse json without long thread blocking if can
	const json = json_parse( buffer )

	// Print json to console
	console.log( json )
}

// Spawn fiber
Fiber.run( log_config )
```

## Compatibility

- [Async functions](#async-functions)
- [Gracefull degradation](#gracefull-degradation)

### Async functions

```typescript
// Async in fiber
function main() {
	const promise = fetch( '//example.org' )
	const response = Fiber.wait( promise )
	const json = Fiber.wait( response.json() )
	console.log( json )
}

const fiber = Fiber.run( main )
```

```typescript
// Fiber in async
async function main2 () {
	console.log( 'start' )
	await fiber
	console.log( 'finish' )
})

main2()
```

### Gracefull degradation

Some features can be transparently disabled if fibers isn't supported.

```typescript
// Continue in next tick if possible
function tick() {

	// Do nothing if fibers isnt' supported
	if( typeof Fiber !== 'function' ) return

	// Do nothing if called outside any fiber
	if( !Fiber.current ) return

	// Wait few time
	Fiber.wait( new Promise( process.nextTick ) )

}
```

## Limitations

**Usage of global variables and `finally` block together should be changed to support execution in fibers**

```typescript
let context = null

function foo( callback1, callback2 ) {

	const prev = {}
	context = {}

	try {
		 // If `Fiber.wait` is called other `foo` execution can change context
		callback1()
		
		// Then we get wrong context there
		callback2()
		
	} finally {
		context = prev
	}
	
}
```

```typescript
const context = new WeakMap

function foo( callback1, callback2 ) {

	// Every context is fiber bound
	const prev = context.get( Fiber.current )
	context.set( Fiber.current , {} )

	try {
		 // Other `foo` execution don't change our context
		callback1()
		
		// Then we get right context there
		callback2()
		
	} finally {
		context.set( Fiber.current , prev )
	}
	
}
```

## Related Implementations

- [node-fibers](https://github.com/laverdet/node-fibers) - [NodeJS](https://nodejs.org/) native extension that adds fibers to v8 runtime.
- [f-promise](https://github.com/Sage/f-promise) - Wrapper around `node-fibers` with API like proposed.
- [$mol_fiber](https://github.com/eigenmethod/mol/tree/master/fiber) - [VanillaJS](http://vanilla-js.com/) fibers implementation based on restarts.
- [Suspense API](https://github.com/facebook/react/pull/12279) in [ReactJS](https://reactjs.org/) based on restarts.
- [Cancellation](http://bluebirdjs.com/docs/api/cancellation.html) in [BlueBird](http://bluebirdjs.com) - Cancellation API in promises.
- [Meteor](https://www.meteor.com/) - based on `node-fibers` popular web-framework.

## Other Languages

- [Greenlets](https://greenlet.readthedocs.io/) in [Python](https://www.python.org/)
- [Goroutines](https://tour.golang.org/concurrency/1) in [Go](https://golang.org/)
- [Fibers](http://ddili.org/ders/d.en/fibers.html) in [D](http://dlang.org/)
- [Fibers proposal](http://cr.openjdk.java.net/~rpressler/loom/Loom-Proposal.html) for [JVM](https://java.com/)

## Stackfull vs stackless coroutines

- [What Color is Your Function?](https://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/)
- [Lightweight Threads for Concurrency](https://mnotes.me/dev/2019/01/13/coroutines.html)
- [Different async js implementations of same app](https://github.com/nin-jin/async-js)
