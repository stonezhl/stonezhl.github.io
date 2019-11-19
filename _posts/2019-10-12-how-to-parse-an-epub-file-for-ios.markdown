---
layout: post
title: "How to Parse an ePub File for iOS"
categories: ios
---

Several months ago, I built an ePub parser from the ground up. During that time, I gained a much greater understanding about XML parsing and the ePub specifications. I would like to share what I have learned with you.

## Getting Started

Before starting, let’s figure out what an ePub file is?

![epub_structure_diagram](/assets/epub_structure_diagram.png)

An ePub file is an e-book file format, which is a compressed folder. The file formats typically contained in an ePub file are XML (including OPF & NCX), HTML, CSS, etc. The XML files contain the data structure of the ePub format. The other file types are mainly used for rendering e-book pages. Today we will focus on parsing XML files to extract useful data, especially the OPF file, which stores the metadata and resource structure of an e-book. Also, we need to parse container.xml to obtain the exact location of the OPF file.

Based on the understanding of an ePub file, we can list our implement steps:
- Uncompress an ePub file
- Parse the container.xml file to get the path of the OPF file
- Parse the OPF file to extract the data we want

## Uncompress EPUB File

To unzip an ePub file, I am going to use a third-party framework, [ZIPFoundation](https://github.com/weichsel/ZIPFoundation). Instead of wrapping [minizip](https://github.com/nmoinvaz/minizip), it uses libcompression library that provided by Apple. According to this [blog](https://thomas.zoechling.me/journal/2017/07/ZIPFoundation.html) of its author, libcompression has a better performance on decoding, which is exactly what we need.

Talk is cheap. Show me the code.

``` swift
import ZIPFoundation

class EPUBParser {
    static func parseFile(at sourceURL: URL) -> EPUBBook? {
        // Get the ePub's filename
        let filename = sourceURL.deletingPathExtension().lastPathComponent
        // The directory to save the unzipped files
        let destinationURL = URL(fileURLWithPath: NSTemporaryDirectory(), isDirectory: true).appendingPathComponent(filename)
        // Get the file manager
        let fileManager = FileManager()
        // Check if already unzipped
        if fileManager.fileExists(atPath: destinationURL.path) == false {
            do {
                // Make sure the directory exists. if not, create one
                try fileManager.createDirectory(at: destinationURL, withIntermediateDirectories: true, attributes: nil)
                // Unzip the ePub file
                try fileManager.unzipItem(at: zipFileURL, to: destinationURL)
            } catch {
                print("Extraction of the ePub file failed with error: \(error)")
                return nil
            }
        }
        // Parse the ePub file
        return EPUBBook(contentsOf: destinationURL)
    }
}
```

## XML Parser Selection

As mentioned above, the main job is extracting data from these XML files, so it’s necessary to choose an appropriate XML parser before continuing. First of all, let's review some of the commonly used XML concepts.

### SAX vs. DOM

There are two ways to parse XML file: SAX and DOM. Both of them have their pros and cons, so understanding is the first step to find our XML parsing library.
- **SAX parser** is event-based. When it is going through the XML file, will keep sending events in sequence. Each event is relevant to tag, attribute, or text in the element, such as startDocument, endDocument, startElement, endElement, foundCharacters, etc. It is good at handling large file with limit memory.
- **DOM parser** is tree-based. It will load the entire document and build up a DOM tree in memory to let you query any element easily. It is good at querying data from complicated format.

The OPF file format is tiny and very complicated, so the DOM parser seems to be our best choice for this scenario.

### Hash Query vs. XPath/XQuery/CSS Selector vs. Decoder

Now, that we have chosen to use the DOM parser, we need to consider how to query the elements. 
- **Hash Query** is more like using a chain of keys to access the element you want. It’s so easy to use that you even don’t need any tutorial. Thinking about the way [SwiftJSON](https://github.com/SwiftyJSON/SwiftyJSON) parsing a JSON file, they are similar.
- **XPath/XQuery/CSS Selector** is a popular way in the Selenium community. Using one string, you can do lots of things more than you thought but needs some learning if you are not familiar with these.
- **Decoder** works like [JSONDecoder](https://developer.apple.com/documentation/foundation/jsondecoder). You write a mapping in a model and then everything will go automatically. The only thing you need to do is accessing the corresponding property.

### Selection

Here is a list of representational libraries:

|Framework	|Native	|Processing	|Query	|Note	|
|---	|---	|---	|---	|---	|
|**[XMLParser](https://developer.apple.com/documentation/foundation/xmlparser)**	|Y	|SAX	|-	|Available for iOS & macOS	|
|---	|---	|---	|---	|---	|
|[**XMLDocument**](https://developer.apple.com/documentation/foundation/xmldocument)	|Y	|DOM	|XPath / XQuery	|Only available for macOS	|
|**[libxml2](http://xmlsoft.org/)**	|Y	|SAX & DOM	|XPath / CSS Selector	|C library, so need write a wrapper	|
|[**Kanna**](https://github.com/tid-kijyun/Kanna)	|N	|SAX & DOM	|XPath / CSS Selector 	|Built on libxml2	|
|[**SWXMLHash**](https://github.com/drmohundro/SWXMLHash)	|N	|DOM	|Hash Query	|Built on XMLParser	|
|[**XMLParsing**](https://github.com/ShawnMoore/XMLParsing)	|N	|DOM	|Decoder	|Built on XMLParser	|

Considering the OPF file is very complicated and very flexible, I prefer to use **Kanna** with **XPath**.

### Little More

This article is not going to consider benchmark and Objective C version libraries. If you want to learn more, please check another article was written by [Ray Wenderlich](https://twitter.com/rwenderlich): “[XML Tutorial for iOS: How To Choose The Best XML Parser for Your iPhone Project](https://www.raywenderlich.com/3145-xml-tutorial-for-ios-how-to-choose-the-best-xml-parser-for-your-iphone-project)”

## Parse the Container File

You could always find the **container.xml** in **META-INF** folder depending on this [container section](https://www.w3.org/Submission/2017/SUBM-epub-ocf-20170125/#sec-container-metainf-container.xml). Here is a sample:

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<container xmlns="urn:oasis:names:tc:opendocument:xmlns:container" version="1.0">
    <rootfiles>
        <rootfile full-path="epub/content.opf" media-type="application/oebps-package+xml"/>
    </rootfiles>
</container>
```

The only thing we care about is `full-path="epub/content.opf"`, so we are going to write the first XPath query to get the path of the OPF file.

``` swift
import Foundation

class EPUBBook {
    let container: ContainerDocument
    
    init?(contentsOf baseURL: URL) {
        // Find the location of container.xml
        let containerURL = baseURL.appendingPathComponent("META-INF/container.xml")
        // Parse the container file
        guard let container = ContainerDocument(url: containerURL) else { return nil }
        self.container = container
        // Get the location of the OPF file
        let opfURL = baseURL.appendingPathComponent(container.opfPath)
    }
}
```

``` swift
import Kanna

struct ContainerDocument {
    let opfPath: String

    let document: XMLDocument

    init?(url: URL) {
        do {
            // Just one line to get the DOM document of a XML file
            document = try Kanna.XML(url: url, encoding: .utf8)
            // Create namespace for the 'container' element
            let namespace = ["ctn": "urn:oasis:names:tc:opendocument:xmlns:container"]
            // Our first XPath query
            let xpath = "//ctn:rootfile[@full-path]/@full-path"
            // Execute the XPath query by 'at_xpath'
            guard let path = document.at_xpath(xpath, namespaces: namespace)?.text else { return nil }
            opfPath = path
        } catch {
            print("Parsing the XML file at \(url) failed with error: \(error)")
            return nil
        }
    }
}
```

Since most code can be understood by comments quickly, here I will focus on explaining two parts in the struct **ContainerDocument**: the XML namespace and our first XPath query.

``` swift
let namespace = ["ctn": "urn:oasis:names:tc:opendocument:xmlns:container"]
```

**XML namespaces** are used for providing uniquely named elements and attributes in an XML document. In this container.xml, `<container xmlns="urn:oasis:names:tc:opendocument:xmlns:container" version="1.0">` presents the element **container** and its children elements are in the namespace `xmlns="urn:oasis:names:tc:opendocument:xmlns:container"`, so we need to create a namespace for the element **rootfile**. It doesn't matter what you are going to name the key, while I am using **ctn**. Please keep in mind you have to use the same key later in the XPath query.

``` swift
let xpath = "//ctn:rootfile[@full-path]/@full-path"
```

Let’s crack the XPath query based on [XPath syntax](https://www.w3schools.com/xml/xpath_syntax.asp):
- `"//rootfile"` selects all the **rootfile** elements from the current node **containerDocument** no matter where they are.
- `"//ctn:rootfile"` means only getting the **rootfile** elements in the namespace **ctn**.
- `"//ctn:rootfile[@full-path]"` makes sure all the elements must have a **full-path** attribute.
- `"//ctn:rootfile[@full-path]/@full-path"` gets the **full-path** attribute of the selected elements.

``` swift
let path = document.at_xpath(xpath, namespaces: namespace)?.text
```

In Kanna, there are two main methods for executing XPath query:
- **at_xpath** will return only one XMLElement which match the XPath query
- **xpath** will return an array of XMLElement which match the XPath query

Here we only need to find one **full-path** and it is supposed to be only one, so we are using method **at_xpath**.

## Understand the OPF File

After getting the path of the OPF file, the next and last step is parsing it! All the code will be very similar to parsing the container file, except the format specification is much more complicated. Let’s be patient and enjoy the happiness coming with XPath. Let’s see what an OPF file looks like:

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<package xmlns="http://www.idpf.org/2007/opf" dir="ltr" prefix="se: http://standardebooks.org/vocab/1.0" unique-identifier="uid" version="3.0" xml:lang="en-US">    
    <metadata xmlns:dc="http://purl.org/dc/elements/1.1/">
        <!--The DCMES elements-->
        <!--The meta elements-->
        <!--The link elements-->
    </metadata>    
    <manifest>
        <!--The item elements-->
    </manifest>
    <spine toc="ncx">
        <!--The itemref elements-->
    </spine>
    <!--The collection elements-->
    <!--The guide elements-->
    <!--Other elements-->
</package>
```

In an OPF file, there is a root element [**package**](https://www.w3.org/Submission/2017/SUBM-epub-packages-20170125/#sec-package-def), which has three main children elements: **metadata**, **manifest**, and **spine**. Here is the code to parse the **package** element:

``` swift
import Foundation

class EPUBBook {
    // ...
    let opf: OPFDocument

    init?(contentsOf baseURL: URL) {
        // ...

        // Parse the OPF file
        guard let opf = OPFDocument(url: opfURL) else { return nil }
        self.opf = opf
    }
}
```

``` swift
import Kanna

struct OPFDocument {
    let metadata: OPFMetadata
    let manifest: OPFManifest
    let spine: OPFSpine
    // Other elements, like collection, guide, etc.

    let document: XMLDocument

    init?(url: URL) {
        do {
            // Get the OPF XMLDocument
            document = try Kanna.XML(url: url, encoding: .utf8)
            // Create namespace for the 'package' element
            let opfNamespace = ["opf": "http://www.idpf.org/2007/opf"]
            // Execute XPath query to fetch the 'package' element
            guard let package = document.at_xpath("/opf:package", namespaces: opfNamespace) else { return nil }
            // Parse the three main elements
            // Will explain them later
            guard let metadata = OPFMetadata(package: package) else { return nil }
            guard let manifest = OPFManifest(package: package) else { return nil }
            guard let spine = OPFSpine(package: package) else { return nil }
            self.metadata = metadata
            self.manifest = manifest
            self.spine = spine
        } catch {
            print("Parsing the XML file at \(url) failed with error: \(error)")
            return nil
        }
    }
}
```

Let's focus on the struct **OPFDocument**:
- The namespace `"http://www.idpf.org/2007/opf"` could be found in the **package** element.
- `"/opf:package"` selects all the **package** elements from the root node **opfDocument** in the namespace **opf**.
- Using method **at_xpath** to return only one XMLElement, because there should be exactly one **package** element.

## The Metadata Element

[The metadata element](https://www.w3.org/Submission/2017/SUBM-epub-packages-20170125/#sec-pkg-metadata) contains three main type elements: the **DCMES** elements, the **meta** elements, and the **link** elements. We will focus on the **DCMES** elements instead of explaining them all. Otherwise, you need at least several hours to finish this article. Here I am going to pick some elements as an example. For more detail, please check [DCMES required elements](https://www.w3.org/Submission/2017/SUBM-epub-packages-20170125/#sec-opf-dcmes-required) and [DCMES optional elements](https://www.w3.org/Submission/2017/SUBM-epub-packages-20170125/#sec-opf-dcmes-optional).

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<package xmlns="http://www.idpf.org/2007/opf" dir="ltr" prefix="se: http://standardebooks.org/vocab/1.0" unique-identifier="uid" version="3.0" xml:lang="en-US">
    <metadata xmlns:dc="http://purl.org/dc/elements/1.1/">
        <!--The DCMES elements-->
        <dc:identifier id="uid">url:https://standardebooks.org/ebooks/lewis-carroll/alices-adventures-in-wonderland</dc:identifier>
        <dc:title id="title">Alice's Adventures in Wonderland</dc:title>
        <dc:language>en-GB</dc:language>
        <dc:creator id="author">Lewis Carroll</dc:creator>
        <dc:publisher id="publisher">Standard Ebooks</dc:publisher>
        <dc:date>2015-05-12T00:01:00Z</dc:date>
        <!--The meta elements-->
        <meta content="cover.jpg" name="cover"/>
        <meta property="dcterms:modified">2017-03-09T17:21:15Z</meta>
        <!--The link elements-->
        <link href="onix.xml" rel="onix-record"/>
        <!--Other elements-->
    </metadata>
    <!--Other elements-->
</package>
```

All the **DCMES** elements will be in the namespace **dc**. The statement of this namespace is in the **metadata** element `<metadata xmlns:dc="http://purl.org/dc/elements/1.1/">`. Here is the code to fetch the **DCMES** elements we need:

``` swift
import Kanna

struct OPFMetadata {
    // DCMES Required Elements
    private(set) var identifiers = [String]()
    private(set) var titles = [String]()
    private(set) var languages = [String]()
    // DCMES Optional Elements
    private(set) var creators = [String]()
    private(set) var date: String?
    private(set) var publisher: String?
    // Others like contributor, description, etc.

    init?(package: XMLElement) {
        let opfNamespace = ["opf": "http://www.idpf.org/2007/opf"]
        guard let metadata = package.at_xpath("opf:metadata", namespaces: opfNamespace) else { return nil }
        // DCMES
        let dcNamespace = ["dc": "http://purl.org/dc/elements/1.1/"]
        let dcmes = metadata.xpath("dc:*", namespaces: dcNamespace)
        for dc in dcmes {
            guard let text = dc.text else { continue }
            switch dc.tagName {
            case "identifier":
                identifiers.append(text)
            case "title":
                titles.append(text)
            case "language":
                languages.append(text)
            case "creator":
                creators.append(text)
            case "date":
                date = text
            case "publisher":
                publisher = text
            default:
                break
            }
        }
    }
}
```

We have two simple XPath queries here:
- `"opf:metadata"` selects all the **metadata** elements that are children of the current node **package** in the namespace **opf**.
- Using method **at_xpath** selects exactly one **metadata** element.
- `"dc:*"` selects all the elements that are children of the **metadata** elements in the namespace **dc**. Maybe you already noticed this time we use method **xpath** to get an array of elements, instead of fetching only one.

## The Manifest Element

[The manifest element](https://www.w3.org/Submission/2017/SUBM-epub-packages-20170125/#sec-pkg-manifest) provides exhaustive locations and types of all the resource files. It should look like below:

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<package xmlns="http://www.idpf.org/2007/opf" dir="ltr" prefix="se: http://standardebooks.org/vocab/1.0" unique-identifier="uid" version="3.0" xml:lang="en-US">
    <manifest>
        <item href="toc.ncx" id="ncx" media-type="application/x-dtbncx+xml"/>
        <item href="text/uncopyright.xhtml" id="uncopyright.xhtml" media-type="application/xhtml+xml"/>
        <item href="toc.xhtml" id="toc.xhtml" media-type="application/xhtml+xml" properties="nav"/>
        <item href="text/chapter-2.xhtml" id="chapter-2.xhtml" media-type="application/xhtml+xml"/>
        <item href="images/cover.jpg" id="cover.jpg" media-type="image/jpeg" properties="cover-image"/>
        <item href="text/chapter-3.xhtml" id="chapter-3.xhtml" media-type="application/xhtml+xml"/>
        <item href="images/titlepage.png" id="titlepage.png" media-type="image/png"/>
        <item href="text/chapter-1.xhtml" id="chapter-1.xhtml" media-type="application/xhtml+xml"/>
        <!--Other items-->
    </manifest>
    <!--Other elements-->
</package>
```

For each item element, there are several attributes:
- **id** attribute will be used for identifying the specific item element.
- **href** attribute shows the location of the particular item element.
- **media-type** attribute presents the media type of the file.
- **properties** attribute will provide more information like it is a navigation file or cover image file.

``` swift
import Kanna

struct OPFManifest {
    private(set) var items = [String: ManifestItem]()

    init?(package: XMLElement) {
        let opfNamespace = ["opf": "http://www.idpf.org/2007/opf"]
        guard let manifest = package.at_xpath("opf:manifest", namespaces: opfNamespace) else { return nil }
        let itemElements = manifest.xpath("opf:item", namespaces: opfNamespace)
        for itemElement in itemElements {
            guard let item = ManifestItem(itemElement) else { continue }
            items[item.id] = item
        }
    }
}

public struct ManifestItem {
    let id: String
    let href: String
    let mediaType: String
    let properties: String?
    // Others like duration, fallback, etc.

    init?(_ item: XMLElement) {
        guard let itemId = item["id"] else { return nil }
        guard let itemHref = item["href"] else { return nil }
        guard let itemMediaType = item["media-type"] else { return nil }
        id = itemId
        href = itemHref
        mediaType = itemMediaType
        properties = item["properties"]
    }
}
```

This part of code fetches exactly one **manifest** element and makes a dictionary mapping item id to an item so that the **spine** element can find the item quickly by its id. The XPath query `"opf:item"` will select all the **item** elements that are children of the current node **manifest** in the namespace **opf**.

## The Spine Element

[The spine element](https://www.w3.org/Submission/2017/SUBM-epub-packages-20170125/#sec-pkg-spine) defines an ordered list of manifest item references that represent the default reading order of e-book. It should look like below:

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<package xmlns="http://www.idpf.org/2007/opf" dir="ltr" prefix="se: http://standardebooks.org/vocab/1.0" unique-identifier="uid" version="3.0" xml:lang="en-US">
    <spine toc="ncx">
        <itemref idref="titlepage.xhtml"/>
        <itemref idref="chapter-1.xhtml"/>
        <itemref idref="chapter-2.xhtml"/>
        <itemref idref="chapter-3.xhtml"/>
        <itemref idref="uncopyright.xhtml"/>
        <!--Other itemrefs-->
    </spine>
    <!--Other elements-->
</package>
```

All the **itemref** elements are in order, and you can find the corresponding manifest item by the **idref **attribute which is same as manifest item’s **id** attribute. Let’s write code to fetch all the **idref** attributes:

``` swift
import Kanna

 struct OPFSpine {
    private(set) var idrefs = [String]()

    init?(package: XMLElement) {
        guard let spine = package.at_xpath("opf:spine", namespaces: XPath.opf.namespace) else { return nil }
        idrefs = spine.xpath("opf:itemref[@idref]/@idref", namespaces: opfNamespace)
                      .map { $0.text }
                      .compactMap { $0 }
    }
}
```

Feel familiar about these two XPath queries? Let’s crack it again!
- `"opf:spine"` selects all the **spine** elements that are children of the current node **package** in the namespace **opf**.
- `"itemref"` selects all the **itemref** elements that are children of the current node **spine**.
- `"opf:itemref"` means only getting the **itemref** elements in the namespace **opf**.
- `"opf:itemref[@idref]"` makes sure all the elements must have a **idref** attribute.
- `"opf:itemref[@idref]/@idref"` gets the **idref** attribute of the selected elements.

## In The End

Finally, we parsed all the essential data from an ePub file. You could extract more data you want by following [EPUB Package 3.1](https://www.w3.org/Submission/2017/SUBM-epub-packages-20170125/#sec-overview). Also, it will be great to consider the ePub 2.0 specification as well, which is a little different from ePub 3.0.

If you need a playground to practice, feel free to use the unit tests of [Bookbinder](https://github.com/stonezhl/Bookbinder). You could learn the specification from the comments and practice not only XPath query but also CSS Selector by replacing the query code in the classes of group **Parser**.