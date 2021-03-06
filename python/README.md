# [ArcGIS For Developer](https://developers.arcgis.com)

### Get token to access ArcGIS APIs (if you have an API key skip this)
#### Step 1: Get API Token
* Create an account to access [ArcGIS For Developer](https://developers.arcgis.com/dashboard)
* go to [signup](https://developers.arcgis.com/sign-up/)
* if you have an account login at https://developers.arcgis.com/sign-in/

#### Step 2: creating a token
* Once logged in you can find a temporary token at [ArcGIS For Developer](https://developers.arcgis.com/dashboard)
* You can also create an application and generate tokens for it.
* Refer to [Route service with synchronous execution](https://developers.arcgis.com/rest/network/api-reference/route-synchronous-service.htm) to get an in depth idea of various post request 
  parameters.
  
#### Step 3: Getting Geocodes for Source and Destination from ArcGIS
* Use the following code to call ArcGIS API to fetch the geocode of the locations
```python
import json
import requests

#API key for Esri-Arcgis-Maps
Token_Esri=os.environ.get('Esri-Arcgis-Maps_API_Key')

def get_geocodes_from_arcgis(address): 
    url="https://geocode.arcgis.com/arcgis/rest/services/World/GeocodeServer/findAddressCandidates" 
    para={'f': 'json' , 'singleLine' : address , 'outFields' : 'Match_addr,Addr_type'}
    longitude,latitude=requests.get(url,params=para).json()['candidates'][0]['location'].values()
    return(longitude,latitude)     #note it returns long lat
```
With this in place, make a POST request: https://route.arcgis.com/arcgis/rest/services/World/Route/NAServer/Route_World/solve

#### Step 4: Extracting polyline from ArcGIS using Source-Destination Geocodes

With `payload attributes` with following keys

```
    payload = {
        "type":"features",
        "features":  [
            {
                "geometry": {                #source coordinates
                             "x": source_longitude,
                             "y": source_latitude,
                             }
                },
            {
                "geometry": {               #destination coordinates
                             "x": destination_longitude, 
                             "y": destination_latitude
                             }
                }
            ]
        }
```

### Note:
You should see full path as series of coordinates, we convert it to
`polyline`

```python
import polyline as poly
response_from_arcgis=requests.post(arcgis_url,data = {'f': 'json','token': token_Esri,'stops':json.dumps(payload)}).json()
coordinate_list=[i[::-1] for i in response_from_arcgis['routes']['features'][0]['geometry']['paths'][0]]
polyline_from_arcgis=poly.encode(coordinate_list)
```
* Use the following function to get polyline from ArcGIS
```python
def get_polyline_from_arcgis(source_longitude,source_latitude,destination_longitude,destination_latitude):
    #url to post request
    arcgis_url = "https://route.arcgis.com/arcgis/rest/services/World/Route/NAServer/Route_World/solve"
    #prepare payload in similar structure and update feature coordinates 
    payload = {
        "type":"features",
        "features":  [
            {
                "geometry": {                #source coordinates
                             "x": source_longitude,
                             "y": source_latitude,
                             }
                },
            {
                "geometry": {               #destination coordinates
                             "x": destination_longitude, 
                             "y": destination_latitude
                             }
                }
            ]
        }
    #response file after post to the given link using payload as value for stop and providing other parameters
    response_from_arcgis=requests.post(arcgis_url,data = {'f': 'json','token': Token_Esri,'stops':json.dumps(payload)},timeout=200).json()
    #making a list for all coordinates to make polyline NOTE ARCGIS provides lon-lat pairs but we need lat-lon pairs
    coordinate_list=[i[::-1] for i in response_from_arcgis['routes']['features'][0]['geometry']['paths'][0]]
    #Encoding coordinate lists into polyline
    polyline_from_arcgis=poly.encode(coordinate_list)
    return(polyline_from_arcgis)
```

Note:

We extracted the polyline for a route from ArcGIS Routing API.

We need to send this route polyline to TollGuru API to receive toll information

## [TollGuru API](https://tollguru.com/developers/docs/)

### Get key to access TollGuru polyline API
* create a dev account to receive a [free key from TollGuru](https://tollguru.com/developers/get-api-key)
* suggest adding `vehicleType` parameter. Tolls for cars are different than trucks and therefore if `vehicleType` is not specified, may not receive accurate tolls. For example, tolls are generally higher for trucks than cars. If `vehicleType` is not specified, by default tolls are returned for 2-axle cars. 
* Similarly, `departure_time` is important for locations where tolls change based on time-of-the-day.
* Use the following function to get rates from TollGuru.

```python
def get_rates_from_tollguru(polyline):
    #Tollguru querry url
    Tolls_URL = 'https://dev.tollguru.com/v1/calc/route'
    
    #Tollguru resquest parameters
    headers = {
                'Content-type': 'application/json',
                'x-api-key': Tolls_Key
                }
    params = {
                #Explore https://tollguru.com/developers/docs/ to get best of all the parameter that tollguru has to offer 
                'source': "esri",
                'polyline': polyline ,                      # this is the encoded polyline that we made     
                'vehicleType': '2AxlesAuto',                #'''Visit https://tollguru.com/developers/docs/#vehicle-types to know more options'''
                'departure_time' : "2021-01-05T09:46:08Z"   #'''Visit https://en.wikipedia.org/wiki/Unix_time to know the time format'''
                }
    
    #Requesting Tollguru with parameters
    response_tollguru= requests.post(Tolls_URL, json=params, headers=headers).json()
    
    #checking for errors or printing rates
    if str(response_tollguru).find('message')==-1:
        return(response_tollguru['route']['costs'])
    else:
        raise Exception(response_tollguru['message'])
```

The working code can be found in Esri-Arcgis-Maps.py file.

## License
ISC License (ISC). Copyright 2020 &copy;TollGuru. https://tollguru.com/

Permission to use, copy, modify, and/or distribute this software for any purpose with or without fee is hereby granted, provided that the above copyright notice and this permission notice appear in all copies.

THE SOFTWARE IS PROVIDED "AS IS" AND THE AUTHOR DISCLAIMS ALL WARRANTIES WITH REGARD TO THIS SOFTWARE INCLUDING ALL IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS. IN NO EVENT SHALL THE AUTHOR BE LIABLE FOR ANY SPECIAL, DIRECT, INDIRECT, OR CONSEQUENTIAL DAMAGES OR ANY DAMAGES WHATSOEVER RESULTING FROM LOSS OF USE, DATA OR PROFITS, WHETHER IN AN ACTION OF CONTRACT, NEGLIGENCE OR OTHER TORTIOUS ACTION, ARISING OUT OF OR IN CONNECTION WITH THE USE OR PERFORMANCE OF THIS SOFTWARE.
