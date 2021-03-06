# Plugin integration with Advanced Content Filter

CKEditor consists of a number of {@link CKEDITOR.feature editor features} like
commands, buttons, combo boxes, dialog windows, etc. The idea behind plugins is
to extend the set of available features but since [Advanced Content Filter](#!/guide/dev_advanced_content_filter)
(**introduced by CKEditor 4.1**), features just like a content are subject of filtering.

Advanced Content Filter brings lots of goodies that must be considered when
developing CKEditor plugins. Those impact on a development process and
slightly change the idea behind data processing. With Advanced Content Filter,
plugins can take control over the content available in the editor and adaptively
adjust user interface when allowed content changes. Of all the properties,
these are crucial for correct ACF integration into editor plugins:

* {@link CKEDITOR.feature#property-allowedContent} &mdash; determines
  a type of content that is allowed by the feature to enter the editor &ndash; in most cases
  this is the content that this feature generates.
* {@link CKEDITOR.feature#property-requiredContent} &mdash; defines a minimal
  set of content types that must be enabled to let the feature work.
* {@link CKEDITOR.feature#property-contentForms} &mdash; defines markup transformations
  for consistency and correctness.

This guide is based on [Simple Plugin (Part 2)](#!/guide/plugin_sdk_sample_2)
tutorial and explains all the necessary code that makes it compatible with
Advanced Content Filter.

<p class="tip">
	You can <a href="guides/plugin_sdk_sample_2/abbr2.zip">download the
	entire plugin folder</a> used in Simple Plugin (Part 2) to follow the changes
	made by this guide.
</p>


## Integrating with ACF to introduce the content of a new type

If you start with the code created for [Simple Plugin (Part 2)](#!/guide/plugin_sdk_sample_2)
but **without
specifying {@link CKEDITOR.feature#property-allowedContent allowedContent} or
{@link CKEDITOR.feature#property-requiredContent requiredContent}** you may notice
that indeed the plugin is working but incoming abbreviations are
filtered out as soon as possible. Try setting this HTML in source mode and switch
back to WYSIWYG (you can do this twice) to see that `<abbr>` tag is gone:

	<p>What is <abbr title="Advanced Content Filter">ACF</abbr>?</p>

Why is it so? Editor doesn't know that since the **Abbreviation** feature is enabled (the button
is added to the toolbar), it should accept `<abbr>` tag as an **allowed content**. Without
{@link CKEDITOR.feature#property-allowedContent allowedContent} property specified,
`<abbr>` tags will always be discarded what makes the button useless.

To introduce `<abbr>` tag correctly and automatically, extend the `abbrDialog` command
definition when calling {@link CKEDITOR.dialogCommand#method-constructor CKEDITOR.dialogCommand}
constructor:

	new CKEDITOR.dialogCommand( 'abbrDialog', {
		allowedContent: 'abbr'
	} );

Try inserting our test HTML and check what's going on when switching between
modes. This is what you should see:

	<p>What is <abbr>ACF</abbr>?</p>

But where is the `title` attribute? It's gone. Yes, you have to [specify which
attributes are allowed](#!/guide/dev_allowed_content_rules). We forgot about it
when setting `allowedContent`. Let's fix this:

	new CKEDITOR.dialogCommand( 'abbrDialog', {
		allowedContent: 'abbr[title]'
	} );

Now `title` will be accepted for any `<abbr>` tag. Loading the **Abbreviation** button
will automatically extend filtering rules to accept a new content type and let
the new feature do the work.


### A button, command or maybe a plugin &ndash; which of them is a *feature*?

What does it mean that "the feature is enabled"? Why did we defined the
{@link CKEDITOR.feature#property-allowedContent allowedContent} property
for the **Abbreviation** command definition, not for the button or the entire plugin?

The first thing to notice is that one plugin can introduce many features.
For example the **basicstyles** plugin adds few buttons,
each of them being a single feature. So a button is more likely to be a feature.

However, in most cases a button just triggers a command, for example, our
**Abbreviation** button has a related `abbrDialog` command. This command
may also be triggered by a keystroke (see {@link CKEDITOR.config#keystrokes})
or directly from code (by {@link CKEDITOR.editor#execCommand}). Therefore,
usually a command is the "root" of a feature, but not always &ndash; e.g.
**Format** dropdown does not have a related command, so  the
{@link CKEDITOR.feature#property-allowedContent allowedContent} property
is defined directly on it.

The most typical way of enabling a feature is by adding its button to the
toolbar. Editor handles features activated this way automatically and:

1. it checks if a button has the
{@link CKEDITOR.feature#property-allowedContent allowedContent} property
(is a feature itself); if yes its allowed content rule is registered to
the {@link CKEDITOR.editor#filter} which is responsible for all main ACF functions,
2. if a button is not a feature itself, but it has a related command,
then that command is registered as a feature.

If you need to register a feature manually from your plugin, then
you can use the {@link CKEDITOR.editor#addFeature} method. It accepts
an object implementing the {@link CKEDITOR.feature} interface.
Read the API documentation for more details.


## Integrating with ACF to activate editor features

Once we made the editor to work with the new content type,
this is a good moment to check what would happen if the **Abbreviation** plugin
was enabled but the {@link CKEDITOR.config#allowedContent} deliberately discarded the
`<abbr>` tag:

	CKEDITOR.replace( 'editor1', {
		extraPlugins: 'abbr',
		allowedContent: 'p'	// Only paragraphs will be accepted.
	});

You may notice that many features like formatting buttons, text alignment and
many others are gone; they discover that the content they produce (`<strong>`, `text-align`, etc.)
is invalid within this configuration environment.
All except the **Abbreviation** button, which, in fact, also makes no longer sense because
user configuration overwrites any rules
[automatically added](#!/guide/plugin_sdk_integration_with_acf-section-2) by the feature.

By specifying {@link CKEDITOR.feature#property-requiredContent requiredContent} property
in a command definition, we make sure that **Abbreviation** button will adaptively
adjust to filtering rules set by the user:

	new CKEDITOR.dialogCommand( 'abbrDialog', {
		allowedContent: 'abbr[title]',
		requiredContent: 'abbr'
	} );

Rule `requiredContent: 'abbr'` means that the **Abbreviation** button requires
`<abbr>` tag to be enabled to work. Otherwise, the feature will be disabled.
This makes sense because inserting and managing abbreviations in an editor that
discards this kind of content is pointless.

Now let's consider another configuration that brings our plugin back to life by
accepting `<abbr>` back again:

	CKEDITOR.replace( 'editor1', {
		extraPlugins: 'abbr',
		allowedContent: 'p abbr'
	});

Rule `allowedContent: 'p abbr'` means that any attribute will be striped out
of `<abbr>` tag, including `title`. Still the **Abbreviation** plugin provides a dialog
window for both editing abbreviations (tag contents) and explanations (`title`).
However the second field is no longer necessary once `title` is discarded.

With Advanced Content Filtering we can specify which dialog fields are
enabled and which are not, depending on filtering rules set in the config. Let's
do this by modifying explanation field in the dialog definition
(`plugins/abbr/dialogs/abbr.js`):

	elements: [
		...
		{
			type: 'text',
			id: 'title',
			label: 'Explanation',
			requiredContent: 'abbr[title]',	// Title must be allowed to enable this field.
			...
		}
		...
	]

But wait. There's an **Advanced Settings** tab in the dialog that may be used for
setting `id` attributes. Remember what our configuration (`allowedContent: 'p abbr'`)
says: "paragraphs and abbreviations, no attributes". **Abbreviation** plugin
should consider this fact and disable the **Advanced Settings** tab unless `id`
is allowed:

	contents: [
		...
		{
			id: 'tab-adv',
			label: 'Advanced Settings',
			requiredContent: 'abbr[id]',	// ID must be allowed to enable this field.
			elements: [
				...
			]
		}
		...
	]

This is it. Let's see how the **Abbreviation** dialog changes with different
`config.allowedContent`:

`allowedContent = 'p abbr'` | `allowedContent = 'p abbr[id]'` | `allowedContent = 'p abbr[title,id]'`
:------------------: | :------------:  | :------------:
{@img acfDialog2.png} | {@img acfDialog1.png} | {@img acfDialog0.png}


## Integrating with ACF for content transformations

Advanced Content Filter introduces
[content transformations](#!/guide/dev_advanced_content_filter-section-4)
that help to clean-up HTML and make it consistent. We can use this functionality
in the **Abbreviation** feature automatically to convert invalid `<acronym>` tag into `<abbr>`.
Basically to do this, the `contentForms` property must be defined that
determines which tag is correct and accepted by the editor with the highest priority:

	new CKEDITOR.dialogCommand( 'abbrDialog', {
		allowedContent: 'abbr[title]',
		requiredContent: 'abbr',
		contentForms: [
			'abbr',
			'acronym'
		]
	} );

This configuration implies the following transformation:

	// HTML before
	<p>What is <acronym title="Advanced Content Filter">ACF</acronym>?</p>

	// HTML after
	<p>What is <abbr title="Advanced Content Filter">ACF</abbr>?</p>

Editor will automatically convert `<acronym>` tags into `<abbr>` when pasting contents
and editing source code.

Read more about {@link CKEDITOR.feature#contentForms contentForms}.

<p class="tip">
	You can also <a href="guides/plugin_sdk_integration_with_acf/abbr3.zip">download the
	whole plugin folder</a> inluding the icon and the fully commented source code.
</p>

## Further reading

* [Advanced Content Filter guide](#!/guide/dev_advanced_content_filter).
* {@link CKEDITOR.filter} &mdash; the main class responsible for ACF features.
* {@link CKEDITOR.feature} &mdash; an interface representing editor feature; used in combination with {@link CKEDITOR.filter#addFeature}.