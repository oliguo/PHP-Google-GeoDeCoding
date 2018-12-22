# php_Google_GeoDeCoding
Decoding the latitude, longitude to the location address by Google geocode API

## Database Structure(MySQL)

### Location Main Table
```sql
CREATE TABLE `location_sync_area` (
  `location_sync_area_id` int(255) NOT NULL,
  `lsa_mix_unique` varchar(255) NOT NULL,
  `lsa_lg` varchar(255) NOT NULL DEFAULT 'zh-TW' COMMENT 'zh-TW,en',
  `country` varchar(255) NOT NULL,
  `administrative_area_level_1` varchar(255) NOT NULL,
  `administrative_area_level_2` varchar(255) NOT NULL,
  `neighborhood` varchar(255) NOT NULL,
  `lsa_date` varchar(255) NOT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

ALTER TABLE `location_sync_area`
  ADD PRIMARY KEY (`location_sync_area_id`),
  ADD UNIQUE KEY `mix_unique_id` (`lsa_mix_unique`);
  
ALTER TABLE `location_sync_area`
  MODIFY `location_sync_area_id` int(255) NOT NULL AUTO_INCREMENT;
COMMIT;
```

### Location Logging Table
```sql
CREATE TABLE `location_sync` (
  `location_sync_id` int(255) NOT NULL,
  `ls_lat` float(10,6) NOT NULL,
  `ls_lng` float(10,6) NOT NULL,
  `ls_google_raw` longtext NOT NULL,
  `ls_lg` varchar(255) NOT NULL DEFAULT 'zh-TW' COMMENT 'zh-TW,en',
  `formatted_address` text NOT NULL,
  `country` varchar(600) NOT NULL,
  `administrative_area_level_1` varchar(600) NOT NULL,
  `administrative_area_level_2` varchar(600) NOT NULL,
  `neighborhood` varchar(600) NOT NULL,
  `route` varchar(600) NOT NULL,
  `street_number` varchar(600) NOT NULL,
  `premise` varchar(600) NOT NULL,
  `ls_date` varchar(255) NOT NULL,
  `location_sync_area_id` int(255) NOT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

ALTER TABLE `location_sync`
  ADD PRIMARY KEY (`location_sync_id`);
  
ALTER TABLE `location_sync`
  MODIFY `location_sync_id` int(255) NOT NULL AUTO_INCREMENT;
COMMIT;
```

## Google Geocode API

[Developer Guide](https://developers.google.com/maps/documentation/geocoding/intro)

## PHP Function Program

### Decoding function from API 
```php
function location_sync_google($lat,$lng){
    $post_url = "https://maps.googleapis.com/maps/api/geocode/json?address=" . $lat.",".$lng . "&key=Your_API_Key&language=en";//zh-TW
    $post_parameter_array = [];
    $ch = curl_init();
    $options = array(
        CURLOPT_URL => $post_url,
        CURLOPT_HEADER => 0,
        CURLOPT_VERBOSE => 0,
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_USERAGENT => "Mozilla/4.0 (compatible;)",
        CURLOPT_POST => true,
        CURLOPT_POSTFIELDS => http_build_query($post_parameter_array),
    );
    curl_setopt_array($ch, $options);
    curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, false); 
    curl_setopt($ch, CURLOPT_SSL_VERIFYHOST, 0);  
    $result = curl_exec($ch);
    curl_close($ch);
    $res = json_decode($result, true);
    //print_r($res);
    $data = [
        "status"=>false,
        "ls_lat" => $lat,
        "ls_lng" => $lng,
        "ls_google_raw"=>$result,
        "ls_lg"=>"en",
        "formatted_address"=>"",
        "country"=>"",
        "administrative_area_level_1"=>"",
        "administrative_area_level_2"=>"",
        "neighborhood"=>"",
        "route"=>"",
        "street_number"=>"",
        "premise"=>"",
        "ls_date"=>date("Y-m-d H:i:s"),
        "location_sync_area_id"=>""
    ];
    if (count($res) > 0) {
        if (isset($res['status'])) {
            if (strtolower($res['status']) === 'ok') {
                if (isset($res['results'])) {
                    if (count($res['results']) > 0) {
                        $data['status']=true;
                        foreach($res['results'] as $item){
                            if(isset($item['formatted_address'])){
                                if(empty($data['formatted_address'])){
                                   $data['formatted_address']=$item['formatted_address'];
                                }
                            }
                            if(isset($item['address_components'])){
                                foreach($item['address_components'] as $address){
                                    if(!empty($address['types'][0])&&(!empty($address['long_name']))){
                                        if(empty($data['country'])){
                                            if($address['types'][0]==='country'){
                                                $data['country']=$address['long_name'];
                                            }
                                        }
                                        if(empty($data['administrative_area_level_1'])){
                                            if($address['types'][0]==='administrative_area_level_1'){
                                                $data['administrative_area_level_1']=$address['long_name'];
                                            }
                                        }
                                        if(empty($data['administrative_area_level_2'])){
                                            if($address['types'][0]==='administrative_area_level_2'){
                                                $data['administrative_area_level_2']=$address['long_name'];
                                            }
                                        }
                                        if(empty($data['neighborhood'])){
                                            if($address['types'][0]==='neighborhood'){
                                                $data['neighborhood']=$address['long_name'];
                                            }
                                        }
                                        if(empty($data['route'])){
                                            if($address['types'][0]==='route'){
                                                $data['route']=$address['long_name'];
                                            }
                                        }
                                        if(empty($data['street_number'])){
                                            if($address['types'][0]==='street_number'){
                                                $data['street_number']=$address['long_name'];
                                            }
                                        }
                                        if(empty($data['premise'])){
                                            if($address['types'][0]==='premise'){
                                                $data['premise']=$address['long_name'];
                                            }
                                        }
                                    }
                                }
                            }
                            
                        }
                    }
                }
            }
        }
    }
    return $data;
}
```

### SQL Function(with mysqli to handle database for sample)
```php
function location_sync_insert($conn,$data=[]){
    $sql="insert into location_sync("
            . "ls_lat, "
            . "ls_lng, "
            . "ls_google_raw, "
            . "ls_lg, "
            . "formatted_address, "
            . "country, "
            . "administrative_area_level_1, "
            . "administrative_area_level_2, "
            . "neighborhood, "
            . "route, "
            . "street_number, "
            . "premise, "
            . "ls_date,"
            . "location_sync_area_id) values("
            . "'". htmlspecialchars($data['ls_lat'], ENT_QUOTES)."',"
            . "'". htmlspecialchars($data['ls_lng'], ENT_QUOTES)."',"
            . "'". htmlspecialchars($data['ls_google_raw'], ENT_QUOTES)."',"
            . "'". htmlspecialchars($data['ls_lg'], ENT_QUOTES)."',"
            . "'". htmlspecialchars($data['formatted_address'], ENT_QUOTES)."',"
            . "'". htmlspecialchars($data['country'], ENT_QUOTES)."',"
            . "'". htmlspecialchars($data['administrative_area_level_1'], ENT_QUOTES)."',"
            . "'". htmlspecialchars($data['administrative_area_level_2'], ENT_QUOTES)."',"
            . "'". htmlspecialchars($data['neighborhood'], ENT_QUOTES)."',"
            . "'". htmlspecialchars($data['route'], ENT_QUOTES)."',"
            . "'". htmlspecialchars($data['street_number'], ENT_QUOTES)."',"
            . "'". htmlspecialchars($data['premise'], ENT_QUOTES)."',"
            . "'". htmlspecialchars($data['ls_date'], ENT_QUOTES)."',"
            . "'". htmlspecialchars($data['location_sync_area_id'], ENT_QUOTES)."'"
            . ")";
    if(mysqli_query($conn, $sql)){
        return mysqli_insert_id($conn);
    }else{
        return false;
    }
}

function location_sync_area_insert($conn,$data=[]){
    $mix=htmlspecialchars($data['ls_lg'])
            .'-'.htmlspecialchars($data['country'])
            .'-'.htmlspecialchars($data['administrative_area_level_1'])
            .'-'.htmlspecialchars($data['administrative_area_level_2'])
            .'-'.htmlspecialchars($data['neighborhood']);
    $sql="insert into location_sync_area("
            . "lsa_mix_unique,"
            . "lsa_lg, "
            . "country, "
            . "administrative_area_level_1, "
            . "administrative_area_level_2, "
            . "neighborhood, "
            . "lsa_date) values("
            . "'". md5($mix)."',"
            . "'". htmlspecialchars($data['ls_lg'], ENT_QUOTES)."',"
            . "'". htmlspecialchars($data['country'], ENT_QUOTES)."',"
            . "'". htmlspecialchars($data['administrative_area_level_1'], ENT_QUOTES)."',"
            . "'". htmlspecialchars($data['administrative_area_level_2'], ENT_QUOTES)."',"
            . "'". htmlspecialchars($data['neighborhood'], ENT_QUOTES)."',"
            . "'". htmlspecialchars($data['ls_date'], ENT_QUOTES)."'"
            . ")";
    //echo $sql;
    if(mysqli_query($conn, $sql)){
        return mysqli_insert_id($conn);
    }else{
        return false;
    }
}

function location_sync_area_checking($conn,$data=[]){
    $sql="select location_sync_area_id from location_sync_area "
            . "where "
            . "lsa_lg='". htmlspecialchars($data['ls_lg'], ENT_QUOTES)."' "
            . "and country='". htmlspecialchars($data['country'], ENT_QUOTES)."' "
            . "and administrative_area_level_1='". htmlspecialchars($data['administrative_area_level_1'], ENT_QUOTES)."' "
            . "and administrative_area_level_2='". htmlspecialchars($data['administrative_area_level_2'], ENT_QUOTES)."' "
            . "and neighborhood='". htmlspecialchars($data['neighborhood'], ENT_QUOTES)."' ";
    //echo $sql;
    $result= mysqli_query($conn, $sql);
    if($result){
        while($row= mysqli_fetch_assoc($result)){
            return $row['location_sync_area_id'];
        }
        return false;
    }else{
        return false;
    }
}
```

### Run Sample Function
```php
$data=location_sync_google($post['lat'],$post['lng']);
if($data['status']){
    $location_sync_area_id=location_sync_area_checking($conn,$data);
    if(!$location_sync_area_id){
        $location_sync_area_id=location_sync_area_insert($conn,$data);
        if(!$location_sync_area_id){
            return false;
        }
    }
    $data['location_sync_area_id']=$location_sync_area_id;
    return location_sync_insert($conn,$data);
}else{
    return false;
}
```

### Data Preview of Location Main Table

| location_sync_area_id | lsa_mix_unique                   | lsa_lg | country   | administrative_area_level_1 | administrative_area_level_2 | neighborhood | lsa_date            |
|-----------------------|----------------------------------|--------|-----------|-----------------------------|-----------------------------|--------------|---------------------|
| 1                     | F89D359A47A9B908F3D53411D8FCA0F9 | en     | Hong Kong | Kowloon                     | Wong Tai Sin District       | San Po Kong  | 2018-12-12 10:00:00 |



