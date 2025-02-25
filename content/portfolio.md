# Demos

there are obviously plenty of other things I've worked in
these are just the publicly-hosted web apps
I've also worked on a formal verification frameworks, a _, a _, and many, many Haskell libraries
some of this can be seen on my GitHub

## source

GH

for the sake of completeness, not everything public I've done is on GH
https://gitlab.haskell.org/ghc/ghc/-/merge_requests/12758
wx

## Monpad

<div id="monpad-grid">

<div id="monpad-buttons">
  <button onclick='window.setMonpadLayout("default")'>
    default layout
  </button>
  <button onclick='window.setMonpadLayout("full")'>
    full gamepad layout
  </button>
  <button onclick='window.setMonpadLayout("mouse")'>
    mouse layout
  </button>
  <button onclick='window.sendMonpadUpdate({"ShowElement": "Red"})'>
    show red button
  </button>
  <button onclick='window.sendMonpadUpdate({"HideElement": "Red"})'>
    hide red button
  </button>
</div>

<iframe
  id="monpad"
  title="Monpad"
  width="400"
  height="200"
  src="/monpad.html?username=George"
>
</iframe>

<div class="wrapper">
  <pre id="monpad-layout"></pre>
</div>

<div class="wrapper">
  <pre id="monpad-output"></pre>
</div>

<script>
const maxLines = 10
const outputElement = document.getElementById("monpad-output")
const layoutElement = document.getElementById("monpad-layout")
document.addEventListener("monpad-client-update", e => {
  outputElement.textContent = outputElement.textContent.split("\n")
    .slice(-maxLines+1).join("\n")
    + (outputElement.textContent ? '\n' : '') + e.detail
})
window.resolveRelativeURL = s => new URL(s, document.URL + "/")
window.sendMonpadUpdate = detail => document.dispatchEvent(new CustomEvent("monpad-server-update", {detail}))
window.setMonpadLayout = s => {
  window.fetch(resolveRelativeURL(`monpad/layouts/${s}.dhall`)).then(r => r.text().then(t => {
    layoutElement.textContent = t
  }))
  // TODO eventually Monpad will have a Haskell Wasm frontend which will support Dhall input directly
  // then we can support actual freeform user input, and use `SetLayout` instead of `SwitchLayout`
  window.sendMonpadUpdate({"SwitchLayout": `${s}`})
}
window.setMonpadLayout("default")
</script>

</div>

## primer

backend solid, frontend lacking polish and being rewritten
should be just about intuitive enough to anyone who's used to functional programming
[my Zurihac talk](https://www.youtube.com/watch?v=TLcLY5W2cTo)

## fourmolu live

I didn't actually work on the web demo itself
to my embarrassment, this is almost certainly the most impactful thing I've developed
I implemented all the initial key options, and the defaults are essentially my personal preferences
355 GH stars and the one project of mine I often see mentioned in the wild (though rarely spelt correctly...)
I've since stepped back a lot due to limited time and ultimately not actually caring much about formatting
https://github.com/fourmolu/fourmolu/issues/112#issuecomment-1185518058

## pretty-simple

a very widely-used Haskell library,
of which I became co-maintainer in 2020 after rewriting about half of it (i.e. the printer but not the parser)
I implemented this web demo
yes, styling etc. could be better - see GH issue

## more to come...

Primer Haskell
THE

I've been heavily exploring using GHC's Wasm backend with Miso, and converting some of my old command-line utilities in to web apps for a less technical audience

<!--

### picture-click-from-svg

it's very simple, but as far as I know,
this might even be the first useful publicly-hosted software to be made using GHC's WASM backend
actually not true (ormolu+fourmolu) - maybe first with Miso or proper FFI?

### svgone

similar vibe to above
i.e. a prototype of a tool which would be useful to semi-technical users, and most accessible through a web interface -->
