# Tampering with legal documents with Azure Documents Intelligence

December 16, 2024

Sensitive transactions execution often requires to show proofs of ID and proofs of ownership: this requirements is regulated in many industries, and it is part of a broader process called Know Your Customer (KYC).

Agents (customs officers, bank clerks, accountants, ...) must painstakingly type-in the identification fields into their back-end systems, causing both processing delays and significant human workload.

## Meet Azure AI Document Intelligence

To expedite this task, and to save their customers' money, Microsoft proposes an Azure AI service called Document Intelligence ("DocIntel"), where said customers can rely on a range of pre-existing data extraction templates (invoices, passports, tax forms, ...) or create their own ones.

When a template is selected, images of legal/commercial/formal documents are sent by the customer to the DocIntel API endpoint and analyzed with computer vision.

The relevant fields (which depend on the model) are extracted and assigned a confidence score by the service. They are returned to the customer in JSON format.

To ease the template-modeling experience, a Studio is provided as part of DocIntel. Once an image is uploaded in Azure Portal, the left-hand pane displays the image itself, and the right-hand pane displays extracted data and confidence scores:


Testing Malaysian passport data extraction using the Azure AI Doc Intel Studio.

<!-- image -->

## Staging a WYSIWYG attack against Doc Intelligence

Suppose we can craft a special image, so that we break the all-important "What You See Is What You Get" (WYSIWIG) paradigm ?

- What You See : the image of a legal document featuring a set of identification and validity data.
- What You Get : a JSON containing another set of identification and validity data, with a very high confidence score.

If we could stage such an attack, here are a few real-world implications this could entail:

- 1. Modifying the destination country of a money transfer, from "Spain" to a blacklisted country like "North Corea";
- 2. Swaping the origins of a cattle shipment, from "Venezuela" to "Poland";
- 3. Abusing an expired country residence document, by modifying its expiration date;
- 4. Forging an identify to obtain a SIM card for performing SMS phishing campaigns
- 5. ...

We could perform these many variants of ID spoofing without even using the studio: as long as our KYC application is enslaved to the Azure DocIntel API, the trick could work.

How could we achieve that ?

Looking at the documentation, the DocIntel API supports various image formats like JPEG or PNG. Their intended use is to upload a static scan of a document, but formats like PNG have an option to embed multiple frames. These are called Animated PNG (APNG).

Imagine one embeds two slightly different versions of a drivers license

into the same APNG: one for Chris Smith, and another one for Bob Smart. Only the name change. How will the DoctIntel API react ?

<!-- image -->

Chris' driver license

Chris' edited license, now showing Bob Small

<!-- image -->

If the endpoint is hardened, it should block attempts to use APNG instead of PNG, because there is no reason to upload anything else than static scan of images. Multiple frames are a source of confusion and should return a HTTP client error (4**), or an HTTP server error (5**). But we shouldn't expect the typical HTTP 200/202 response codes indicating that the data was processed.

In Procreate (or another image making tool), let's build an APNG made of 2 frames: Chris and Bob's licenses. We set the loop duration to a value as low as possible.

To optimize the rendering, we can use an APNG assembler tool that will remove the loop and that will make one of the frames invisible. Let's call one of the license frame00.png, the other license frame01.png, and use

apngasm, an animated PNG assembler

<!-- image -->

The -l1 option sets the loop count to 1, and the -f makes one of the frames invisible.

Now our APNG looks exactly like a normal PNG, but it is not !

## Proof-Of-Concept

Let's put our experience to the ultimate test. We submit spoofed.png to the API endpoint, and...

- 1. Bingo ! We get an HTTP 202 response. DocIntel supports animated PNGs :-)
- 2. We can control which identity we want the API to process, by setting the proper frame in the APNG.
- 3. Azure's extraction confidence of the faked identity is extremely high

The API endpoint accepts animated PNG !

<!-- image -->

We can get a visual confirmation of this exploit in the Studio:

Getting Chris' identifiers when we set Bob as the visible frame !

<!-- image -->

Bob's ID shows up on the left, but Chris' ID shows up on the right... With 97% + confidence.

## Outcome

Interfacing a KYC application with the Azure DocIntel APIs opens up an opportunity for hackers to inject spoofed IDs into customers backends.

This proof-of-concept highlights an emergent risk with multimedia AIs: What You See Is NOT What You Get.

The reason is that computer vision processes files, whereas human vision processes retina imprints.

If you are using an OCR as part of your KYC process, I am urging you to perform an image format validation step to block animated images .

## Responsible disclosure timeline

'''
08/Nov/2024: The vulnerability was successfully exploited and Microsoft Security Response Center (MSRC) was notified

15/Nov/2024: Issue was resubmitted to MSRC with full details

09/Dec/2024: After reviewing the finding, MSRC decided to close the case because the "reported behavior is not a vulnerability as the correct values are displayed accurately".

10/Dec/2024: Disclosure report (this document) was shared with MSRC

16/Dec/2024: Public disclosure
'''
