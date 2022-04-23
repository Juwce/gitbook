---
description: Tested in Unreal Engine version 4.27
---

# Responding to Unreal Engine Editor Selection (C++)

![This actor responds to being selected and deselected (and moved - see bonus section!) in the Editor. Written in C++. ](../.gitbook/assets/d761e5df55131c7b58888dd5efa272f5.gif)

Want to respond to your Actor being selected or deselected in the Unreal Engine Editor with C++? Read on!

## Problem

I ran across this problem when implementing a volume that I wanted to randomly spawn actors inside of. I wanted to be able to see a sample of possible spawn locations directly in the editor before play began. I came across [this forum post](https://forums.unrealengine.com/t/event-when-actor-selected-in-editor/358904/3) explaining how to respond to editor selection, but unfortunately the solution proposed in that post only works when actors are selected in the viewport, and NOT if that actor is selected in the World Outliner.

## Solution

This is an updated answer to the Unreal Forum post mentioned above that works when the actor gets selected/deselected in the viewport OR the world outliner.

In the Actor's constructor, bind the selection events like this:

```cpp
#include "Engine/Selection.h"

AExampleActor::AExampleActor()
{
    #if WITH_EDITOR
    // Broadcast whenever the editor selection changes (viewport or world 
    // outliner)
    USelection::SelectionChangedEvent.AddUObject(this, 
                                         &ExampleActor::OnSelectionChanged);
    #endif
}
```

Then declare and implement the `OnSelectionChanged` function:

In `ExampleActor.h`

```cpp
#if WITH_EDITOR
    void OnSelectionChanged(UObject* NewSelection);
    bool bSelectedInEditor;
#endif
```

In `ExampleActor.cpp`

```cpp
void ATPickupSpawnVolume::OnSelectionChanged(UObject* NewSelection)
{
	TArray<ExampleActor*> SelectedExampleActors;
	
	// Get ExampleActors from the selection
	USelection* Selection = Cast<USelection>(NewSelection);
	if (Selection != nullptr)
	{
		Selection->GetSelectedObjects<ExampleActor>(
						SelectedExampleActors);
	}
	
	// Search the selection for this actor
	for (ExampleActor* SelectedExampleActor : SelectedExampleActors)
	{
		// If our actor is in the selection and was not previously
		// selected, then this selection change marks the actor
		// being selected
		if (SelectedExampleActor == this && !bSelectedInEditor)
		{
			// Respond to this actor being selected
			bSelectedInEditor = true;
		}
	}

	// If our record shows our actor is selected, but IsSelected() is false,
	// this selection change marks the actor being deselected
	if (bSelectedInEditor && !IsSelected())
	{
		// Respond to this actor being deselected
		bSelectedInEditor = false;
	}
}
#endif
```

And that's it! You can now write whatever code you want to run whenever your actor is selected or deselected in the editor.

## Bonus: Responding to Actor Moved in Editor

This one is much simpler! All AActors have a native `PostEditMove()`function that gets called whenever the actor is moved in the editor.

In `ExampleActor.h`

```cpp
#if WITH_EDITOR
    virtual void PostEditMove(bool bFinished) override;
#endif
```

In `ExampleActor.cpp`

```cpp
#if WITH_EDITOR
void ExampleActor::PostEditMove(bool bFinished)
{
	Super::PostEditMove(bFinished);

	if (bFinished)
	{
		// finished moving (e.g. let go of mouse click)
	}
	else
	{
		// move in progress (e.g. user dragging mouse)
	}
}
#endif
```

## Conclusion

With the above code, you should now be able to add functionality to any of your C++ actors in the editor when they are selected, deselected, or moved. You can use this to display useful information right in the editor, before play begins. Cheers!

![](../.gitbook/assets/d761e5df55131c7b58888dd5efa272f5.gif)
