nginx-mongo-logger(1) -- Pipe Nginx log output into a MongoDB collection
========================================================================

## SYNOPSIS

`nginx-mongo-logger log_file mongo_collection [mongo_host] [mongo_port]`

## DESCRIPTION

Spawns a tail process and listens for complete lines of input. For each line, `nginx-mongo-logger` parses
CLF (Common Log Format) and extracts nuggets of info, which it then inserts into a MongoDB collection.

## LOG FORMAT

The log format that `nginx-mongo-logger` expects is as follows:

	IP_ADDR - USER [TIME_STAMP] "HTTP_METHOD REQUEST_URI HTTP/1.1" STATUS SIZE "REFERRER_URI" "USER_AGENT"

Where `TIME_STAMP` looks like: `14/Oct/2011:19:53:31 -0400`

## MONGODB SETUP

For best results, you should setup a capped collection on your MongoDB server ahead of time. This takes
care of automatic rotation, as old log data will be automatically replaced by new.

	$ mongo
	> use my_database
	> db.createCollection('nginx_logs', { capped: true, size: 32000000 })

## UPSTART JOB

See `examples/upstart.conf` for an example of an Upstart job using `nginx-mongo-logger`.

## ANALYZING YOUR LOGS

Analyzing your stored logs is crazy simple. The easiest way is to write a MapReduce function and store it on the server to cut down on typing.
This example function will rank all 404 errors by frequency and store them in a collection called `dead_links`.

	$ mongo
	> use my_database
	> rank_404s = function() {
		return db.runCommand({
			mapReduce: 'access_logs',
			query: { status: 404 },
			out: 'dead_links',
			map: function() {
				emit(this.method + "::" + this.path, { count: 1 });
			},
			reduce: function(key, vals) {
				var result = { count: 0 };
				vals.forEach(function(val) {
					result.count += val.count;
				});
				return result;
			}
		});
	}
	> db.system.js.save({ _id: 'rank_404s', value: rank_404s })
	> db.eval('rank_404s()')
