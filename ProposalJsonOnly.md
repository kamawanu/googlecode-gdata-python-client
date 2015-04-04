# Introduction #

With the release of JSONC there is a more streamlined data format which we now support in `gdata.core.Jsonc`. Not all services yet support this format, but we can still use JSON in these services by converting the v1 JSON to XML.

This is just an exploratory idea at this point.

# Example Implementation #

The `client` would need to request `json` as the output format for CRUD operations. The JSON data would be edited locally and when it comes time to POST or PUT to a server, the library will convert the v1 JSON object to Atom XML.

Here is an example that I whipped up which shows one way that this conversion could be accomplished.

```
import simplejson
from elementtree import ElementTree


NAMESPACE_ALIASES = {
  'openSearch': 'http://a9.com/-/spec/opensearchrss/1.0/',
  'gd': 'http://schemas.google.com/g/2005',
  'gCal': 'http://schemas.google.com/gCal/2005'
}

def json_to_xml_item(json_item, xml_tag):
  """The json_item must be an object/dict {}."""
  tree = ElementTree.Element(xml_tag)
  for name, value in json_item.iteritems():
    if name == '$t':
      tree.text = value
    elif isinstance(value, (str, unicode)):
      # This should become an XML attribute.
      tree.attrib[name] = value
    elif isinstance(value, (list, tuple)):
      # This is a repeating XML element.
      for item in value:
        xml_tag = json_name_to_xml_tag(name)
        tree.append(json_to_xml_item(item), xml_tag)
    else:
      # This must be an item, so append.
      xml_tag = json_name_to_xml_tag(name)
      tree.append(json_to_xml_item(value, xml_tag))
  return tree

def json_name_to_xml_tag(json_name):
  if json_name.find('$') == -1:
    return json_name
  else:
    namespace_alias, tag = json_name.split('$')
    if namespace_alias in NAMESPACE_ALIASES:
      return '{%s}%s' % (NAMESPACE_ALIASES[namespace_alias], tag)
    else:
      raise Exception('Invalid namespace alias %s' % namespace_alias)

def json_to_xml_entry(json_dict):
  """Converts the old JSON format used by gdata APIs to gdata XML.

  http://code.google.com/apis/gdata/docs/json.html
  """
  tree = json_to_xml_item(json_dict, '{http://www.w3.org/2005/Atom}entry')
  return ElementTree.tostring(tree)



if __name__ == '__main__':
  print 'simple tests'
  print json_to_xml_entry({'title': {'type': 'text', '$t': 'hi title'}})
  print json_to_xml_entry(
      {'title': {'type': 'text', '$t': 'hi title'},
       "gCal$timezone": {"value": "America/Los_Angeles"},
       "gd$eventStatus": {"value": "http://schemas.google.com/g/2005#event.confirmed"},
       "openSearch$startIndex": {"$t": "1"}})

```