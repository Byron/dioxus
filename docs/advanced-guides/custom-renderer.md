# Custom Renderer

Dioxus is an incredibly portable framework for UI development. The lessons, knowledge, hooks, and components you acquire over time can always be used for future projects. However, sometimes those projects cannot leverage a supported renderer or you need to implement your own better renderer.

Great news: the design of the renderer is entirely up to you! We provide suggestions and inspiration with the 1st party renderers, but only really require implementation of the `RealDom` trait for things to function properly.

## The specifics:

Implementing the renderer is fairly straightforward. The renderer needs to:

1. Handle the stream of edits generated by updates to the virtual DOM
2. Register listeners and pass events into the virtual DOM's event system
3. Progress the virtual DOM with an async executor (or disable the suspense API and use `progress_sync`)

Essentially, your renderer needs to implement the `RealDom` trait and generate `EventTrigger` objects to update the VirtualDOM. From there, you'll have everything needed to render the VirtualDOM to the screen.

Internally, Dioxus handles the tree relationship, diffing, memory management, and the event system, leaving as little as possible required for renderers to implement themselves.

For reference, check out the WebSys renderer as a starting point for your custom renderer.

## Trait implementation

The current `RealDom` trait lives in `dioxus_core/diff`. A version of it is provided here:

```rust
pub trait RealDom {
    // Navigation
    fn push_root(&mut self, root: RealDomNode);

    // Add Nodes to the dom
    fn append_child(&mut self);
    fn replace_with(&mut self);

    // Remove Nodes from the dom
    fn remove(&mut self);
    fn remove_all_children(&mut self);

    // Create
    fn create_text_node(&mut self, text: &str) -> RealDomNode;
    fn create_element(&mut self, tag: &str, namespace: Option<&str>) -> RealDomNode;

    // Events
    fn new_event_listener(
        &mut self,
        event: &str,
        scope: ScopeIdx,
        element_id: usize,
        realnode: RealDomNode,
    );
    fn remove_event_listener(&mut self, event: &str);

    // modify
    fn set_text(&mut self, text: &str);
    fn set_attribute(&mut self, name: &str, value: &str, is_namespaced: bool);
    fn remove_attribute(&mut self, name: &str);

    // node ref
    fn raw_node_as_any_mut(&self) -> &mut dyn Any;
}
```

This trait defines what the Dioxus VirtualDOM expects a "RealDom" abstraction to implement for the Dioxus diffing mechanism to function properly. The Dioxus diffing mechanism operates as a [stack machine](https://en.wikipedia.org/wiki/Stack_machine) where the "push_root" method pushes a new "real" DOM node onto the stack and "append_child" and "replace_with" both remove nodes from the stack. When the RealDOM creates new nodes, it must return the `RealDomNode` type... which is just an abstraction over u32. We strongly recommend the use of `nohash-hasher`'s IntMap for managing the mapping of `RealDomNode` (ids) to their corresponding real node. For an IntMap of 1M+ nodes, an index time takes about 7ns which is very performant when compared to the traditional hasher.

## Event loop

Like most GUIs, Dioxus relies on an event loop to progress the VirtualDOM. The VirtualDOM itself can produce events as well, so it's important that your custom renderer can handle those too.

The code for the WebSys implementation is straightforward, so we'll add it here to demonstrate how simple an event loop is:

```rust
pub async fn run(&mut self) -> dioxus_core::error::Result<()> {
    // Push the body element onto the WebsysDom's stack machine
    let mut websys_dom = crate::new::WebsysDom::new(prepare_websys_dom().first_child().unwrap());
    websys_dom.stack.push(root_node);

    // Rebuild or hydrate the virtualdom
    self.internal_dom.rebuild(&mut websys_dom)?;

    // Wait for updates from the real dom and progress the virtual dom
    while let Some(trigger) = websys_dom.wait_for_event().await {
        websys_dom.stack.push(body_element.first_child().unwrap());
        self.internal_dom
            .progress_with_event(&mut websys_dom, trigger)?;
    }
}
```

It's important that you decode the real events from your event system into Dioxus' synthetic event system (synthetic meaning abstracted). This simply means matching your event type and creating a Dioxus `VirtualEvent` type. Your custom event must implement the corresponding event trait:

```rust
fn virtual_event_from_websys_event(event: &web_sys::Event) -> VirtualEvent {
    match event.type_().as_str() {
        "keydown" | "keypress" | "keyup" => {
            struct CustomKeyboardEvent(web_sys::KeyboardEvent);
            impl dioxus::events::KeyboardEvent for CustomKeyboardEvent {
                fn char_code(&self) -> usize { self.0.char_code() }
                fn ctrl_key(&self) -> bool { self.0.ctrl_key() }
                fn key(&self) -> String { self.0.key() }
                fn key_code(&self) -> usize { self.0.key_code() }
                fn locale(&self) -> String { self.0.locale() }
                fn location(&self) -> usize { self.0.location() }
                fn meta_key(&self) -> bool { self.0.meta_key() }
                fn repeat(&self) -> bool { self.0.repeat() }
                fn shift_key(&self) -> bool { self.0.shift_key() }
                fn which(&self) -> usize { self.0.which() }
                fn get_modifier_state(&self, key_code: usize) -> bool { self.0.get_modifier_state() }
            }
            VirtualEvent::KeyboardEvent(Rc::new(event.clone().dyn_into().unwrap()))
        }
        _ => todo!()
```

## Custom raw elements

If you need to go as far as relying on custom elements for your renderer - you totally can. This still enables you to use Dioxus' reactive nature, component system, shared state, and other features, but will ultimately generate different nodes. All attributes and listeners for the HTML and SVG namespace are shuttled through helper structs that essentially compile away (pose no runtime overhead). You can drop in your own elements any time you want, with little hassle. However, you must be absolutely sure your renderer can handle the new type, or it will crash and burn.

For example, the `div` element is (approximately!) defined as such:

```rust
struct div(NodeBuilder);
impl<'a> div {
    #[inline]
    fn new(factory: &NodeFactory<'a>) -> Self {
        Self(factory.new_element("div"))
    }
    #[inline]
    fn onclick(mut self, callback: impl Fn(MouseEvent) + 'a) -> Self {
        self.0.add_listener("onclick", |evt: VirtualEvent| match evt {
            MouseEvent(evt) => callback(evt),
            _ => {}
        });
        self
    }
    // etc
}
```

The rsx! and html! macros just use the `div` struct as a compile-time guard around the NodeFactory.

## Compatibility

Forewarning: not every hook and service will work on your platform. Dioxus wraps things that need to be cross-platform in "synthetic" types. However, downcasting to a native type might fail if the types don't match.

There are three opportunities for platform incompatibilities to break your program:

1. When downcasting elements via `Ref.to_native<T>()`
2. When downcasting events via `Event.to_native<T>()`
3. Calling platform-specific APIs that don't exist

The best hooks will properly detect the target platform and still provide functionality, failing gracefully when a platform is not supported. We encourage - and provide - an indication to the user on what platforms a hook supports. For issues 1 and 2, these return a result as to not cause panics on unsupported platforms. When designing your hooks, we recommend propagating this error upwards into user facing code, making it obvious that this particular service is not supported.

This particular code _will panic_ due to the unwrap. Try to avoid these types of patterns.

```rust
let div_ref = use_node_ref(&cx);

cx.render(rsx!{
    div { ref: div_ref, class: "custom class",
        button { "click me to see my parent's class"
            onclick: move |_| if let Some(div_ref) = div_ref {
                panic!("Div class is {}", div_ref.to_native::<web_sys::Element>().unwrap().class())
            }
        }
    }
})

```

## Conclusion

That should be it! You should have nearly all the knowledge required on how to implement your own renderer. We're super interested in seeing Dioxus apps brought to custom desktop renderers, mobile renderer, video game UI, and even augmented reality! If you're interesting in contributing to any of the these projects, don't be afraid to reach out or join the community.