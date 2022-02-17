# Tile upload and data ingest notes

This document summarises a few things to be aware of when massaging data from transforms or Illustrator files prior to upload to the SMP.

## Layer names and detail levels

Martin has made a Plectica map that visually illustrates how `forLayer` values are assembled [here](https://www.plectica.com/maps/PZLC28SP8/edit/S6MWLKKOF). 

Each label from the labels export has a `forLayer` triple (generated by a `layer` attribute in the Illustrator data) that tells the client which lenses/views and zoom/detail levels it should be displayed on. The format of `forLayer` should be as follows:

    <lens-name>@<detail-level-range>

The lens name is pretty simple -- if I want to display my labels under a lens called `distributors`, then I start my `forLayer` value with `distributors`.

The detail level range is a numerical range (or a single value). Zoom levels start at 0 and (usually) go up to 6. If I want my labels to display between zoom levels 4 and 6, then I pass a detail level range of `4-6`

This means a full, valid `forLayer` value would look something like `distributors@4-6`.

If I want my labels to display under *all* lenses/views, all of the time, then I just omit the lens name entirely and have a `forLayer` value that is just `@<detail-level-range>`, so `@4-6` or similar.

**forLayer values must adhere this format!**. If you try and pass something in a different format like `level-1` then the client won't parse it and your labels won't display at all. 

## How to name your layers in Illustrator so as to display the labels correctly

### Views, layer names, and how they are connected

How and where labels display is linked to both layer name and the view configuration of the map in question.

Currently, views are most commonly defined in .ttl files such as [this one](https://github.com/VisualMeaning/transforms/blob/master/content/petusa_defs.ttl). A simple view definition might look like this:

```
vm:stress a vm:View ;
    vm:name "Stressful homeowner experience" ;
    vm:comment "stressful-homeowner-experience" ;
    vm:usesMapTiles "https://opatlas-live.s3.amazonaws.com/exec-ed/20211020/overlays/stress/{z}-{x}-{y}.png#background:transparent" ;
    .
```

This defines a view, `stress`, which can be accessed by navigating to `<root map url>/stress`. This one was taken from Exec Ed so this view should be live at https://eom.ecosystem.guide/maps/exec-ed/stress (if we haven't changed it...)

The view has the following properties:

- A human-readable `name` attribute.
- A `usesMapTiles` attribute that tells the view the S3 location of the tiles that it should load when the view is accessed.
- A `vm:comment` attribute.

It's this last one that controls how labels display, as despite the name `vm:comment` is actually the **lens name** of the labels that should display on this view.

In the above example, loading this view will also load any labels with the lens name `stressful-homeowner-experience`.

So consider the following 3 potential layer name formats:

    stressful-homeowner-experience@0-2

This will display the labels from this layer on the `stress` view at zoom levels 0, 1 and 2. The labels will not display at zoom levels greater than 2, nor will they display on any other view that does not also have a `vm:comment` attribute of `"stressful-homeowner-experience"`.

If we didn't care about displaying labels at specific zoom levels, perhaps because a map only has one level of detail, we could instead have the layer name:

    stressful-homeowner-experience

Omitting the detail levels from the layer name means that the labels from this layer will display at *all* zoom levels of any view with the attribute `vm:comment "stressful-homeowner-experience"`.

Finally, there is the wildcard version described above:

    @0-6

If the lens name is omitted from the layer name, the labels from this layer will display on **every** view in the map, and they'll even display when no view is selected. They are *always* visible.

### What a valid layer name looks like

Any of the three examples above are valid. We run a conversion script on label exports from Illustrator that converts everything to lower case and replaces all spaces with `-`, so labels exported from a layer called `Stressful Homeowner Experience` will be converted to something with a `forLayer` property of `stressful-homeowner-experience`.

If we have a layer name that consists of just a detail level range, we automatically prepend a `@` to the `forLayer` property, so a layer called `4-6` will get converted to `@4-6`.

The conversion script also removes any prefix characters denoting semantic/non-semantic layers (i.e. `-` or `~` characters) and so these are ignored in the examples below - they can be included or not, and it will make no difference to the resulting `forLayer` values.

With that in mind, the following are valid layer names depending on your use case and view configuration.

- `Stressful Homeowner Experience`
- `4-6`
- `@4-6`
- `Stressful Homeowner Experience@4-6`

The following are *not* valid layer names.

- `Stressful Homeowner Experience 4-6`
- `Stressful Homeowner Experience-4-6`
- `-4-6`

Including any separator between the lens name and the detail level range that is not an `@` character is a no go. Similarly, prepending a bare detail level range with anything that's not an `@` character is also not going to fly.

### When to use wildcard labels (with lens name omitted)

Wildcard layer names (`@0-6` and similar) whose labels are always visible are used almost exclusively on maps with a single semantic view. If a second tileset and view are added, the labels will also display on top of that second view. This is useful if that's the intended behaviour (i.e. the underlying tilesets from each view share the same geography so the label positions still make sense, or if the second view adds some supplementary labels to the first view). However, if you have two semantic views with mutually exclusive labels you *cannot* use wildcard layer names. Instead you need to do a bit of additional configuration in the map definitons.

### Maps with two or more semantic views

These are potentially slightly awkward as there's usually no default view configured for a map. This means that when you load the map by visiting the root URL for the map, you're not actually looking at any specific view. What will load are the base tileset for the map (configured in the Django admin panel for the map) along with any wildcard labels that always display (as these will still display even if there's no view currently selected). Again, for a single tileset map this is usually the desired behaviour, but for maps with multiple semantic views we can't use these wildcard labels. All labels must be associated with a specific view, and that means that if we want somebody loading the map for the first time to see any labels we need to configure a default view that gets loaded at the same time that the map is.

The [Bucks 2025](https://eom.ecosystem.guide/maps/bucks-strategy/) map is a good example of this, as all of its labels are linked to specific views and there are no wildcard labels present. If you click the link you should see that instead of ending up at the viewless root map URL of https://eom.ecosystem.guide/maps/bucks-strategy/, you instead get redirected to https://eom.ecosystem.guide/maps/bucks-strategy/_base -- this loads the `_base` view and loads the tileset and labels associated with that view. The `_base` view is defined like this:

```
vm:_base a vm:View ;
    vm:name "Base map" ;
    vm:comment "base" ;
    vm:usesMapTiles "https://opatlas-live.s3.amazonaws.com/nhs-bucks/20211102/tiles/{z}-{x}-{y}.png" ;
    .
```

So it's just a normal view that we happen to have called `_base`, there's nothing special about it -- it will display all labels with a `forLayer` property of `base` (or `base@0-6` or similar), just like any other view with a `vm:comment` attribute set. 

We assign `_base` as the default view when we define the map:

```
<http://visual-meaning.com/rdf/maps/bucks-strategy/ecosystem> a vm:Map;
    vm:defaultView <http://visual-meaning.com/rdf/_base> ;
    vm:itemSidebar "on" ;
    vm:navContent """
...<sidebar nav markdown>...
""" ;
    .
```

The map has a `vm:defaultView` property that points to the `_base` view, and so it's this that gets loaded as the default view when somebody hits the root map URL.

So for maps with multiple semantic views, make sure all of your layer names have a lens name matching a defined view. Then set one of those views as the default view when defining the map. See the [actual Bucks defs file](https://github.com/VisualMeaning/transforms/blob/master/extras/bucks_defs.ttl) for a practical example of Map + View definitions in this scenario.

## Linking labels to stakeholder data from Excel workbooks

Consider a map that displays a set of stakeholder entities. The stakeholders and their attributes -- name, description, any relationships -- are defined in an Excel workbook and converted to RDF triples via a transform. However, the Excel workbook does not contain any information on how and where to display the stakeholders on the map. This information is provided by the labels export from Illustrator, which comes in the form of a .json file which is converted to triples by the `labels.py` script in `map-tools`.

We now have two sets of triples from the labels and the workbook that, when combined, would give us all of the data we need to display the stakeholders properly. We just need to link the two sets of data together so that the model knows they're referring to the same thing.

The default behaviour for the `labels.py` script in `map-tools` is to attach a `forThing` attribute to each labels that points at another IRI. The client uses this to attach labels to other objects in the data model with the `forThing` IRI. The format of the IRI is as follows:

    'vm:_thing_{row[qcontents].as_slug}'

`row[qcontents]` is the name of the stakeholder as represented on the label in Illustrator, and should match the name of the stakeholder in the workbook. The problem is that whatever transform is converting the workbook data into triples is probably formatting the stakeholder IRI like this:

    'vm:_stakeholder_{row[Category / Stakeholder name].as_slug}'

They're typed differently -- the labels IRI has the `_thing` prefix, while the workbook IRI has the `_stakeholder` prefix. The workbook version is more correct, but the labels has to use a more generic prefix because labels aren't just attached to stakeholders, they can go on painpoints, categories or just about anything.

So we need to create a linking reference that connects our `_thing`-typed IRI to our `_stakeholder`-typed one. This is done by inserting an extra triple into the workbook transform:

    'lets': {
        'iri': 'vm:_stakeholder_{row[Category / Stakeholder name].as_slug}',
    },
    'triples': [
        ...
        ('{iri}', 'owl:sameAs',
            'vm:_thing_{row[Category / Stakeholder name].as_slug}'),
    ],

This `owl:sameAs` triple is saying that the IRI `vm:_stakeholder_<stakeholder_name>` from the workbook is the same thing as the `vm:_thing_<stakeholder_name>` IRI from the labels data. When the client follows the `forThing` reference to the object the labels are supposed to be attached to, it will now find the data under `vm:_stakeholder` as it is supposed to.

Note that if you're doing this as part of a multistage transform where the transform data is being constructed as step one and then the labels data is being added as step two, you need to run any transforms containing `owl:sameAs` predicates with the `--no-resolve-same` argument in `sheet-to-triples`. This stops this aliasing being resolved until the second step of the process when the labels are added in.

## Spaces in Illustrator PDF filenames

**2022-02-16 note** - we now have slightly smarter behaviour around syncing tile outputs to S3 that flattens out spaces from the S3 upload URL, so the below is no longer an issue in so far as PDF filenames are concerned. It's still potentially a useful titbit if your tiles aren't loading from S3, though -- if a space has somehow gotten into the URL, that'll be why.

Spaces in filenames tend to drive programmers nuts at the best of times because they need special handling to escape them in code. They're particularly bad for the tile upload process for lenses because we derive S3 folder names for the lens tiles from the source PDF filename. Say we have a lens called `distributors`, which has exported as a file called `~distributors_tiles.pdf` -- the tiles for this will end up being stored in S3 in the following location:

    s3://opatlas-live/mgs/20210928/overlays/distributors/

All well and good. Now say we have a lens called `Mars Inc`, which has exported as `~Mars Inc_tiles.pdf` - this will end up at

    s3://opatlas-live/mgs/20210928/overlays/Mars Inc/

And this is a problem, because when the client tries to load a tile from this bucket it's going to look for it at the following URL:

    https://opatlas-live.s3.eu-west-2.amazonaws.com/mgs/20210928/overlays/Mars Inc/0-0-0.png

But you can't have spaces in URLs! The URL gets truncated to `https://opatlas-live.s3.eu-west-2.amazonaws.com/mgs/20210928/overlays/Mars` and we get a 403 unauthorised response because that location doesn't exist.

We can work around this by doing some conversion of the filename during the tile slicing and exporting (so `Mars Inc` becomes `mars-inc`), but it is rather annoying. So please, **don't put spaces in your Illustrator filenames**. (Which I'm assuming are derived from names inside Illustrator.)