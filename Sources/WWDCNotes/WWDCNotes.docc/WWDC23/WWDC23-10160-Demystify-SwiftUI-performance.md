# Demystify SwiftUI performance

Learn how you can build a mental model for performance in SwiftUI and write faster, more efficient code. We’ll share some of the common causes behind performance issues and help you triage hangs and hitches in SwiftUI to create more responsive views in your app.

@Metadata {
   @TitleHeading("WWDC23")
   @PageKind(sampleCode)
   @CallToAction(url: "https://developer.apple.com/wwdc23/10160", purpose: link, label: "Watch Video (21 min)")

   @Contributors {
      @GitHubUser(RamitSharma991)
   }
}



* SwiftUI makes it easy to build complex, powerful apps, offering a large set of features and complex controls like lists and tables.
* As the app's complexity grows, performance becomes more important. 
* Small issues can get amplified, and code that works well for a prototype might not work as well in production.
* Performance problems start with **Symptoms** like: 
    * slow navigation pushs
    * broken animations
    * a spinning wait cursor on macOS
* Post identifying a performance problem, the first step towards addressing it is to **Measure**.
* After measuring and verifing that the symptom exists, work on identifying its cause.
* **Identification** is tricky, requires intuition as bugs arise from incorrect assumption. 
* After identifying the root cause, fix the issue through Optimization. 
* Re-measure and Re-verify any fix to prevent performance problems after optimization.
* Post verification, if the problem is resolved, break the loop.

## Prerequisites
- Identity (implicit and explicit)
- View lifetime and View Identity.
- [Demystify SwiftUI WWDC21](https://developer.apple.com/wwdc21/10022)

# Dependencies 
* Views form a graph and SwiftUI looks at dependencies when evaluating code.
* Each child view is dependent on the view value that gets produced by its ancestor but  there are other forms of dependencies too. 
* Dynamic properties are a common source of dependencies.
* Explore dependencies:
    * Use `Self._printChanges`
    * Used as the LLDB command `expression Self._printChanges`
    * printChanges, is for debugging only, has a runtime impact
    * to check if a view might have extra dependencies

*Example code continuing from Demystify SwiftUI WWDC21 for scalable Dog image + printChanges*

```swift
struct ScalableDogImage: View {
  @State private var scaleToFill = false
  var dog: Dog

  var body: some View {

    // It is never guaranteed to always exist and may even be removed in a future release,
    // so never submit a call to this method to the app store.
    let _ = Self._printChanges()

    dog.image
      .resizable()
      .aspectRatio(contentMode: scaleToFill ? .fill : .fit)
      .frame(maxHeight: scaleToFill ? 500 : nil)
      .padding(.vertical, 16)
      .onTapGesture {
        withAnimation { scaleToFill.toggle() }
      }
  }
}
```

### View update tips
- Eliminate unnecessary dependencies
- Extract views if needed
- Explore using Observable. 

# Faster view updates
* Symptoms of slow SwiftUI updates: reduced responsiveness, such as hangs and hitches.
* **Hangs**:
    * Delays in responding to interaction, like a view taking a long time to initially appear. 
    * [Use Instruments to identify hangs](https://developer.apple.com/wwdc23/10248)
* **Hitches**:
    * User-perceivable animation issue like a pause during scrolling or skipped frames of an animation.
    * The root causes of hangs and hitches, especially in SwiftUI, are often related.

## Common sources of slow updates
- Dynamic Property instantiation, such as allocating and initializing a state object or initializing state.
- Expensive view body: a dynamic property could be computed from a view's body, making the view expensive to evaluate.
- Make sure to check for expensive string interpolation or operations like data filtering and other work inside of body.
- Slow identification.

```swift

// accessing model.dogs in the body lazily instantiates the object,
// which brings us to the initializer, which fetches the list of dogs.
// As the code comment says, this could take a long time. This is synchronous work.
struct DogRootView: View {
  @State private var model = FetchModel()
    
  var body: some View {
      DogList(model.dogs)
  }
}

@Observable class FetchModel {
  var dogs: [Dog]
    
  init() {
      fetchDogs()
  }
}
    
  func fetchDogs() {
    // Takes a long time
  }
}

// Updated and fixed using task modifier, asynchronous fetch and awaiting.
struct DogRootView: View {
  @State private var model = FetchModel()
    
  var body: some View {
    DogList(model.dogs)
      .task { await model.fetchDogs() }
  }
}

@Observable class FetchModel {
  var dogs: [Dog]
    
  init() {}
    
  func fetchDogs() async {
    // Takes a long time
  }
}
```

## Other hidden sources of work
- String Interpolation: make sure to cache any strings you might need to frequently use.
- Bundle lookup
- Heap allocation

# Identity in List and Table
* support rich features beyond a simple layout, adding selection, swipe actions, reordering support, and more. 
* These are complex, advanced controls, and understanding identity is critical to ensuring they perform well in your app.

## Built in improvements 
- Faster filtering
- Reduced time to show large lists
- Smoother scrolling

## Lists and Tables
- Construction affects performance.
- use identifiers to know what changes occurred to the data.
- Gather IDs eagerly to ensure consistency.

## Why identity matters
- **Animations**: incremental updates to the same view vs a new view
- **Performance**: identifiers are gathered often

## Row identity with Lists and Foreach
- `public struct ForEach<Data, ID, Content>` is the signature for ForEach from SwiftUI.
- ForEach maps a collection of data onto a resulting sequence of views, producing explicit identity for each of its views.
- Lists, it needs to figure out how many rows to display and the identifier for that row.
- List, visits the data collection up front, determining each element's ID.
- The content closure is called to produce each view.
- Rows are created on-demand.
- List uses a composite of the identity and the content to produce a list row.
- The rows created on-demand correlate to the visible region, plus some system-determined buffer for prefetching or accessibility.
- As the view is scrolled, more views become extant.
- Avoid inline filtering.

```swift

// Constant views per element - ✅
List {
  ForEach(dogs) { dog in
    DogCell(dog)
  }
}

// Variable views per element - ⚠️
List {
  ForEach(dogs) { dog in
    if dog.fetchToy == .ball {
      DogCell(dog)
    }
  }
}

// Unknown views per element - ⚠️
List {
  ForEach(dogs) { dog in
    AnyView(...)
  }
}

// Avoid inline filtering - ⚠️
List {
  ForEach(dogs.filter(...)) { dog in
    DogCell(dog)
  }
}

// Cached filter and constant views per element - ✅
List {
  ForEach(tennisBallDogs) { dog in
    DogCell(dog)
  }
}
```

## Tips for ensuring constant counts
- Avoid using `AnyView` in `ForEach` and lopsided conditions.
- Use an explicit stack where appropriate
- Try to flatten nested ForEach constructions

## Table identification 
- TableRow always resolves to a single row.
- Total row count is of `TableRow` instead of views.
- SwiftUI provides a streamlined initializer that allows us to write `ForEach` in data collections and creates the table rows on our behalf.
- While this initializer is new, it back deploys to all previous operating system versions where Table is available.
- It's a simpler construction that enforces a constant number of rows for the ForEach content, which helps with identification performance.
- Explicit Identity in ForEach with Tables:
  - each row gets identified by its value to improve performance.
  - For back deployment, get the old behavior by either mapping over your collection or by explicitly specifying an ID key path.

### Row counts
- Row count in Lists: `Row count = element count x views per element (ensure constant)`
- Row count in Tables: `Row count = element count x TableRows per element (ensure constant)`

## Faster lists and tables 
- Ensure identifiers are inexpensive 
- Consider the number of rows in ForEach content
