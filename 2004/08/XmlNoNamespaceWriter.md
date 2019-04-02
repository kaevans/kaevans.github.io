# XmlNoNamespaceWriter Redux

August 2, 2004

I recently received a comment from DotNetDave, or someone posting as him.  The comment referred to a post I made titled "Removing Namespaces in XML, Security in ASP.NET" in which I updated a link to another post entitled "How do I Remove Namespaces Using XSLT" where I demonstrated how easy it is to create an XmlWriter based class that did not emit XML namespaces called XmlNoNamespaceWriter.  Dave said the writer does not work with serialization.  I knew that it should, so I wrote a quick test:

````csharp
WindowsApplication1.localhost.Account a = new WindowsApplication1.localhost.Account();

a.accountID = 5;
System.Xml.Serialization.XmlSerializer s = new System.Xml.Serialization.XmlSerializer(a.GetType());
System.IO.StringWriter sw = new System.IO.StringWriter();
XmlNoNamespaceWriter writer = new XmlNoNamespaceWriter(sw);
s.Serialize(writer,a);
writer.Flush();
MessageBox.Show(sw.ToString());
writer.Close();
````

Sure enough, it works with serialization just fine.  However, I did notice something odd, and this might have been the basis for Dave’s post (I dunno, he just said it doesn’t work with serialization).  The XML generated is incorrect.  Writing  foo:bar=’blah’ with the old version would result in bar=’blah’, as expected.  Writing xmlns:foo=’urn:foo:bar’ would result in foo=’urn:foo:bar’, which is unacceptable.  Worse, writing an element with an attribute of xmlns=’urn:foo:bar’ could result in the XML namespace declaration being written out as well.  So, my simple couple lines of code need a couple more simple lines of code.

Here is an updated implementation.

````csharp
    public class XmlNoNamespaceWriter : System.Xml.XmlTextWriter 
    {
        bool skipAttribute = false;

        public XmlNoNamespaceWriter(System.IO.TextWriter writer) : base(writer)
        {
        }

        public override void WriteStartElement(string prefix, string localName, string ns)
        {
            base.WriteStartElement(null,localName,null);
        }


        public override void WriteStartAttribute(string prefix, string localName, string ns)
        {
            //If the prefix or localname are "xmlns", don’t write it.
            if(prefix.CompareTo("xmlns") == 0 || localName.CompareTo("xmlns")==0)
            {
                skipAttribute = true;                
            }
            else
            {
                base.WriteStartAttribute(null,localName,null);
            }
        }

        public override void WriteString(string text)
        {
            //If we are writing an attribute, the text for the xmlns
            //or xmlns:prefix declaration would occur here.  Skip
            //it if this is the case.
            if(!skipAttribute)
            {
                base.WriteString(text);
            }
        }

        public override void WriteEndAttribute()
        {
            //If we skipped the WriteStartAttribute call, we have to
            //skip the WriteEndAttribute call as well or else the XmlWriter
            //will have an invalid state.
            if(!skipAttribute)
            {
                base.WriteEndAttribute();
            }
            //reset the boolean for the next attribute.
            skipAttribute = false;
        }


        public override void WriteQualifiedName(string localName, string ns)
        {
            //Always write the qualified name using only the
            //localname.
            base.WriteQualifiedName(localName,null);
        }
    }
````