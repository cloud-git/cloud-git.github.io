# short url

- https://github.com/YOURLS/YOURLS
- https://github.com/cydrobolt/polr

# 实现

实现基本都是int 自增，int to string(mod and div)

# 源码分析

## 1. YOURLS

yourls-api.php 是入口文件。 shorturl 对应处理方法 yourls_api_action_shorturl（functions-api.php）-> yourls_add_new_link(functions.php) -> yourls_get_next_decimal(functions.php) -> (int)yourls_get_option( 'next_id' )

while id++ insert table; 失败id+1继续insert，解决并发问题。

```
$table = YOURLS_DB_TABLE_OPTIONS;
get_row( "SELECT `option_value` FROM `$table` WHERE `option_name` = '$option_name' LIMIT 1" );
```

```
$id = yourls_get_next_decimal();
			$ok = false;
			do {
				$keyword = yourls_int2string( $id );
				$keyword = yourls_apply_filter( 'random_keyword', $keyword, $url, $title );
				if ( yourls_keyword_is_free($keyword) ) {
					if( @yourls_insert_link_in_db( $url, $keyword, $title ) ){
						// everything ok, populate needed vars
						$return['url']      = array('keyword' => $keyword, 'url' => $strip_url, 'title' => $title, 'date' => $timestamp, 'ip' => $ip );
						$return['status']   = 'success';
						$return['message']  = /* //translators: eg "http://someurl/ added to DB" */ yourls_s( '%s added to database', yourls_trim_long_string( $strip_url ) );
						$return['title']    = $title;
						$return['html']     = yourls_table_add_row( $keyword, $url, $title, $ip, 0, time() );
						$return['shorturl'] = YOURLS_SITE .'/'. $keyword;
					}else{
						// database error, couldnt store result
						$return['status']   = 'fail';
						$return['code']     = 'error:db';
						$return['message']  = yourls_s( 'Error saving url to database' );
					}
					$ok = true;
				}
				$id++;
			} while ( !$ok );
```
yourls_keyword_is_free
```
yourls_keyword_is_reserved( $keyword ) or yourls_keyword_is_taken( $keyword )
yourls_keyword_is_reserved 是配置文件
$yourls_reserved_URL = array(
	'porn', 'faggot', 'sex', 'nigger', 'fuck', 'cunt', 'dick',
);

$already_exists = $ydb->get_var( "SELECT COUNT(`keyword`) FROM `$table` WHERE `keyword` = '$keyword';" );
```

之后yourls_int2string，核心流程结束。

### 感兴趣的方法
yourls_check_IP_flood

登录用户 yourls_is_valid_user 和 白名单 YOURLS_FLOOD_IP_WHITELIST 不限制

最后一次请求时间需要大于YOURLS_FLOOD_DELAY_SECONDS
```
$lasttime = $ydb->get_var( "SELECT `timestamp` FROM $table WHERE `ip` = '$ip' ORDER BY `timestamp` DESC LIMIT 1" );
	if( $lasttime ) {
		$now = date( 'U' );
		$then = date( 'U', strtotime( $lasttime ) );
		if( ( $now - $then ) <= YOURLS_FLOOD_DELAY_SECONDS ) {
			// Flood!
			yourls_do_action( 'ip_flood', $ip, $now - $then );
			yourls_die( yourls__( 'Too many URLs added too fast. Slow down please.' ), yourls__( 'Forbidden' ), 403 );
		}
	}

if( !defined( 'YOURLS_FLOOD_DELAY_SECONDS' ) )
	define( 'YOURLS_FLOOD_DELAY_SECONDS', 15 );
```
## 2. polr

框架 https://lumen.laravel.com/，有点面向对象的意思。

public/index.php -> bootstrap/app.php -> app/Http/routes.php -> LinkController@performShorten -> LinkFactory::createLink ->
```
if (env('SETTING_PSEUDORANDOM_ENDING')) {
		// generate a pseudorandom ending
		$link_ending = LinkHelper::findPseudoRandomEnding();
}
else {
		// generate a counter-based ending or use existing ending if possible
		$link_ending = LinkHelper::findSuitableEnding();
}
```
findSuitableEnding
找到最新一条记录，short_url toBase10 ，然后while linkExists +1
```
$link = Link::where('is_custom', 0)
		->orderBy('created_at', 'desc')
		->first();

POLR_BASE=32 # 数据库中进制，展示进制
```

findPseudoRandomEnding
```
$pr_str = str_random(env('_PSEUDO_RANDOM_KEY_LENGTH'));
```

### ?
没看到并发处理。如果两个请求同时找到相同的可用short_url怎么办

# 其他

采用302跳转，临时跳转，统计请求量等
