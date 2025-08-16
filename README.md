<!DOCTYPE html>
<html>
<head>
	<meta http-equiv="content-type" content="text/html; charset=utf-8"/>
	<title></title>
	<meta name="generator" content="LibreOffice 24.8.7.2 (Linux)"/>
	<meta name="created" content="2025-08-16T14:01:15.403203078"/>
	<meta name="changed" content="2025-08-16T14:01:58.589920966"/>
</head>
<body lang="de-DE" link="#000080" vlink="#800000" dir="ltr"><h1 class="western">
Kismet Probe Monitor</h1>
<p>This repo contains <strong>two Python scripts</strong> that work
together to discover Wi-Fi <strong>probe requests</strong> in Kismet
logs while ignoring “known” networks and clients:</p>
<ul>
	<li><p><code class="western">build_known_ssids.py</code> — builds
	a list of <strong>known SSIDs</strong> and <strong>known clients
	(MACs)</strong> from your Kismet logs and writes a single reference
	file.</p></li>
	<li><p><code class="western">monitor_probes.py</code> —
	continuously scans Kismet logs for <strong>probe requests</strong>
	and writes an <strong>aggregate JSON log</strong> for <strong>unknown</strong>
	SSIDs/clients, organized by <strong>source file → SSID →
	clients</strong>.</p></li>
</ul>
<p>Both scripts use <strong>only Python’s standard library</strong>
and support modern Kismet <strong>SQLite</strong> exports (<code class="western">.kismetdb</code>,
<code class="western">.kismet</code>) and the legacy <strong>XML</strong>
export (<code class="western">.netxml</code>).</p>
<hr/>

<h2 class="western">Table of Contents</h2>
<ul>
	<li><p><a href="#requirements">Requirements</a></p></li>
	<li><p><a href="#input-files">Input Files</a></p></li>
	<li><p><a href="#output-files">Output Files</a></p>
	<ul>
		<li><p><a href="#produced-by-build_known_ssidspy">Produced by
		<code class="western">build_known_ssids.py</code></a></p></li>
		<li><p><a href="#produced-by-monitor_probespy">Produced by
		<code class="western">monitor_probes.py</code></a></p></li>
	</ul>
	<li><p><a href="#script-1-build_known_ssidspy">Script 1:
	build_known_ssids.py</a></p>
	<ul>
		<li><p><a href="#what-it-does">What it does</a></p></li>
		<li><p><a href="#cli">CLI</a></p></li>
		<li><p><a href="#usage">Usage</a></p></li>
		<li><p><a href="#how-known-is-detected">How “known” is detected</a></p></li>
	</ul>
	<li><p><a href="#script-2-monitor_probespy">Script 2:
	monitor_probes.py</a></p>
	<ul>
		<li><p><a href="#what-it-does-1">What it does</a></p></li>
		<li><p><a href="#aggregate-json-structure">Aggregate JSON structure</a></p></li>
		<li><p><a href="#cli-1">CLI</a></p></li>
		<li><p><a href="#usage-1">Usage</a></p></li>
		<li><p><a href="#filtering-logic">Filtering logic</a></p></li>
	</ul>
	<li><p><a href="#notes--behaviors">Notes &amp; Behaviors</a></p></li>
	<li><p><a href="#troubleshooting">Troubleshooting</a></p></li>
</ul>
<hr/>

<h2 class="western">Requirements</h2>
<ul>
	<li><p><strong>Python 3.7+</strong></p></li>
	<li><p>No external packages required.</p></li>
	<li><p>Run the scripts from the folder that contains your Kismet
	logs (or point your shell there).</p></li>
</ul>
<hr/>

<h2 class="western">Input Files</h2>
<p>Place one or more of the following Kismet exports in the working
directory:</p>
<ul>
	<li><p><code class="western">*.kismetdb</code> or <code class="western">*.kismet</code>
	— Kismet SQLite databases.</p></li>
	<li><p><code class="western">*.netxml</code> — legacy Kismet XML
	export (optional; used as a fallback).</p></li>
</ul>
<blockquote>The scripts dynamically inspect the database schema and
JSON blobs inside Kismet’s <code class="western">devices</code>
table (and, if needed, other tables) to extract SSIDs/clients.</blockquote>
<hr/>

<h2 class="western">Output Files</h2>
<h3 class="western">Produced by <code class="western">build_known_ssids.py</code></h3>
<ul>
	<li><p><strong><code class="western">known_entities.json</code></strong><br/>
A
	single reference file with <strong>known SSIDs</strong> and <strong>known
	clients</strong> (MAC addresses):</p>
	<pre class="western"><code class="western">{</code>
<code class="western">  </code><code class="western">&quot;ssids&quot;: [&quot;HomeWiFi&quot;, &quot;Cafe_WLAN&quot;, &quot;...&quot;],</code>
<code class="western">  </code><code class="western">&quot;clients&quot;: [&quot;A0:B1:C2:D3:E4:F5&quot;, &quot;00:11:22:33:44:55&quot;, &quot;...&quot;]</code>
<code class="western">}</code></pre></li>
</ul>
<blockquote><strong>Backward-compatibility:</strong> If you already
have a legacy <code class="western">known_ssids.json</code> (a JSON
list or a dict with <code class="western">ssids</code>/<code class="western">clients</code>),
<code class="western">monitor_probes.py</code> can still use it as a
fallback.</blockquote>
<hr/>

<h3 class="western">Produced by <code class="western">monitor_probes.py</code></h3>
<ul>
	<li><p><strong><code class="western">unknown_probes.json</code></strong>
	(aggregate log; <strong>continuously updated</strong>)<br/>
Hierarchical
	structure: <strong>source file → SSID → clients</strong>.<br/>
Example:</p>
	<pre class="western"><code class="western">{</code>
<code class="western">  </code><code class="western">&quot;capture1.kismetdb&quot;: {</code>
<code class="western">    </code><code class="western">&quot;source_mtime&quot;: 1723802330,</code>
<code class="western">    </code><code class="western">&quot;source_size&quot;: 12345678,</code>
<code class="western">    </code><code class="western">&quot;ssids&quot;: {</code>
<code class="western">      </code><code class="western">&quot;Cafe_WLAN&quot;: {</code>
<code class="western">        </code><code class="western">&quot;total_count&quot;: 3,</code>
<code class="western">        </code><code class="western">&quot;first_seen&quot;: &quot;2025-08-16T11:00:03+0200&quot;,</code>
<code class="western">        </code><code class="western">&quot;last_seen&quot;:  &quot;2025-08-16T11:02:41+0200&quot;,</code>
<code class="western">        </code><code class="western">&quot;clients&quot;: {</code>
<code class="western">          </code><code class="western">&quot;A0:B1:C2:D3:E4:F5&quot;: {</code>
<code class="western">            </code><code class="western">&quot;count&quot;: 2,</code>
<code class="western">            </code><code class="western">&quot;first_seen&quot;: &quot;2025-08-16T11:00:03+0200&quot;,</code>
<code class="western">            </code><code class="western">&quot;last_seen&quot;:  &quot;2025-08-16T11:02:41+0200&quot;,</code>
<code class="western">            </code><code class="western">&quot;timestamps&quot;: &quot;2025-08-16T11:00:03+0200,2025-08-16T11:02:41+0200&quot;</code>
<code class="western">          </code><code class="western">}</code>
<code class="western">        </code><code class="western">}</code>
<code class="western">      </code><code class="western">}</code>
<code class="western">    </code><code class="western">}</code>
<code class="western">  </code><code class="western">}</code>
<code class="western">}</code></pre></li>
</ul>
<p><strong>Field meanings</strong></p>
<ul>
	<li><p><code class="western">source_mtime</code>, <code class="western">source_size</code>:
	file metadata captured when the source was scanned.</p></li>
	<li><p><code class="western">total_count</code>: total number of
	probe events for that SSID (from this source).</p></li>
	<li><p><code class="western">first_seen</code>, <code class="western">last_seen</code>:
	ISO-8601 timestamps with local offset.</p></li>
	<li><p><code class="western">clients[MAC].timestamps</code>: <strong>single
	line</strong>, comma-separated list of timestamps for that
	client/SSID pair.</p></li>
</ul>
<hr/>

<h2 class="western">Script 1: <code class="western">build_known_ssids.py</code></h2>
<h3 class="western">What it does</h3>
<p>Scans all <code class="western">.kismetdb</code>/<code class="western">.kismet</code>/<code class="western">.netxml</code>
files in the current directory to collect:</p>
<ul>
	<li><p><strong>Known SSIDs</strong> — beaconed/advertised SSIDs
	(and OWE SSIDs where present).</p></li>
	<li><p><strong>Known clients</strong> — device MACs that clearly
	behave like 802.11 clients (see heuristics below).</p></li>
</ul>
<p>Writes both sets to <strong><code class="western">known_entities.json</code></strong>.</p>
<h3 class="western">CLI</h3>
<pre class="western" style="margin-bottom: 0.5cm"><code class="western">python3 build_known_ssids.py [--output known_entities.json] [--verbose]</code></pre>
<ul>
	<li><p><code class="western">--output</code> (default:
	<code class="western">known_entities.json</code>): output JSON file.</p></li>
	<li><p><code class="western">--verbose</code>: print diagnostic
	messages (counts per file, fallbacks used, etc.).</p></li>
</ul>
<h3 class="western">Usage</h3>
<pre class="western"><code class="western"># from the directory containing your Kismet logs</code>
<code class="western">python3 build_known_ssids.py --verbose</code>
<code class="western"># -&gt; writes known_entities.json (includes ssids + clients)</code></pre><h3 class="western">
How “known” is detected</h3>
<ul>
	<li><p><strong>SSIDs</strong>: Prefer
	<code class="western">dot11.device/dot11.advertised_ssid_map</code>
	(Kismet JSON), fall back to common SSID fields, and to <code class="western">&lt;essid&gt;</code>
	in <code class="western">.netxml</code> (non-cloaked).</p></li>
	<li><p><strong>Clients</strong>: A device is considered a client
	(and its <code class="western">devmac</code> recorded) if its JSON
	indicates probe behavior (e.g., <code class="western">dot11.probed_ssid_map</code>,
	<code class="western">dot11.device.last_probed_ssid_record</code>)
	or other <code class="western">dot11.client.*</code> structures.
	<code class="western">.netxml</code> <code class="western">&lt;wireless-client&gt;&lt;client-mac&gt;</code>
	is also included.</p></li>
</ul>
<p>Hidden/placeholder SSIDs like <code class="western">&lt;hidden&gt;</code>
or null strings are ignored.</p>
<hr/>

<h2 class="western">Script 2: <code class="western">monitor_probes.py</code></h2>
<h3 class="western">What it does</h3>
<p>Continuously monitors the directory for changes in Kismet files,
extracts <strong>probe SSIDs</strong> (the networks clients are
searching for), and updates an aggregate log <strong>only for unknown
items</strong>:</p>
<ul>
	<li><p>An event is <strong>filtered out</strong> if <strong>SSID is
	known</strong> <em>or</em> <strong>client MAC is known</strong> from
	<code class="western">known_entities.json</code> (or the legacy
	file).</p></li>
	<li><p>The aggregate log <code class="western">unknown_probes.json</code>
	is <strong>updated after every change</strong> (atomic write).</p></li>
</ul>
<h3 class="western">Aggregate JSON structure</h3>
<p>Top-level keys are <strong>source filenames</strong>. Each source
has a <code class="western">ssids</code> map; each SSID has <code class="western">clients</code>,
and each client keeps a comma-separated <code class="western">timestamps</code>
string plus counters and first/last seen times.</p>
<blockquote>This puts the <strong>source dataset first</strong> to
avoid repeating the same source metadata across many entries.</blockquote>
<h3 class="western">CLI</h3>
<pre class="western"><code class="western">python3 monitor_probes.py</code>
<code class="western">  </code><code class="western">[--poll 10]</code>
<code class="western">  </code><code class="western">[--entities known_entities.json]</code>
<code class="western">  </code><code class="western">[--known-ssids known_ssids.json]</code>
<code class="western">  </code><code class="western">[--agg-file unknown_probes.json]</code>
<code class="western">  </code><code class="western">[--include-known]</code>
<code class="western">  </code><code class="western">[--verbose]</code></pre>
<ul>
	<li><p><code class="western">--poll</code> (default <code class="western">10</code>):
	seconds between scans.</p></li>
	<li><p><code class="western">--entities</code> (default
	<code class="western">known_entities.json</code>): path to the
	SSID+client reference file written by the build script.</p></li>
	<li><p><code class="western">--known-ssids</code> (fallback): legacy
	reference file (list or dict with optional <code class="western">ssids</code>/<code class="western">clients</code>).</p></li>
	<li><p><code class="western">--agg-file</code> (default
	<code class="western">unknown_probes.json</code>): aggregate output
	file (continuously updated).</p></li>
	<li><p><code class="western">--include-known</code>: <strong>debug</strong>
	switch — do <em>not</em> filter out known SSIDs/clients (useful to
	verify extraction).</p></li>
	<li><p><code class="western">--verbose</code>: print diagnostics
	(counts per source, fallback paths used).</p></li>
</ul>
<h3 class="western">Usage</h3>
<pre class="western"><code class="western"># 1) Build the known entities first</code>
<code class="western">python3 build_known_ssids.py</code>

<code class="western"># 2) Start the monitor (updates unknown_probes.json continuously)</code>
<code class="western">python3 monitor_probes.py --poll 5</code>

<code class="western"># (Optional) Debug: show/log everything, even if known</code>
<code class="western">python3 monitor_probes.py --include-known --verbose --poll 5</code></pre><h3 class="western">
Filtering logic</h3>
<p>An extracted <code class="western">(SSID, client MAC)</code> probe
event is <strong>ignored</strong> if:</p>
<ul>
	<li><p><code class="western">SSID ∈ known_entities[&quot;ssids&quot;]</code>
	<strong>OR</strong></p></li>
	<li><p><code class="western">client MAC ∈
	known_entities[&quot;clients&quot;]</code></p></li>
</ul>
<p>Otherwise, it is <strong>aggregated</strong> under:</p>
<pre class="western"><code class="western">unknown_probes.json</code>
<code class="western">  </code><code class="western">-&gt; &lt;source file&gt;</code>
<code class="western">      </code><code class="western">-&gt; ssids[&lt;SSID&gt;]</code>
<code class="western">         </code><code class="western">-&gt; clients[&lt;client MAC&gt;]</code>
<code class="western">            </code><code class="western">-&gt; timestamps (comma-separated)</code></pre>
<hr/>

<h2 class="western">Notes &amp; Behaviors</h2>
<ul>
	<li><p><strong>Change detection</strong>: Files are re-scanned when
	their <strong>size or mtime</strong> changes. New files are picked
	up automatically.</p></li>
	<li><p><strong>Timestamps</strong>: Local time formatted as ISO-8601
	with offset (e.g., <code class="western">2025-08-16T11:02:41+0200</code>).</p></li>
	<li><p><strong>Atomic writes</strong>: Output JSON files are written
	via a temp file and then replaced to avoid partial writes.</p></li>
	<li><p><strong>Hidden SSIDs</strong>: <code class="western">&lt;hidden&gt;</code>
	and null/zero strings are skipped.</p></li>
	<li><p><strong>Heuristics</strong>: Scripts tolerate multiple Kismet
	schema variants and JSON field layouts (maps vs. lists).</p></li>
	<li><p><strong>Performance</strong>: For very large DBs, increase
	<code class="western">--poll</code> to reduce churn, or run the
	build script on a subset first.</p></li>
</ul>
<hr/>

<h2 class="western">Troubleshooting</h2>
<p><strong><code class="western">known_entities.json</code></strong><strong>
is empty or tiny</strong></p>
<ul>
	<li><p>Ensure your captures contain <strong>beaconing APs</strong>
	(for SSIDs) and/or <strong>client activity</strong> (for known
	clients).</p></li>
	<li><p>Run with <code class="western">--verbose</code> to see which
	tables/columns are inspected.</p></li>
</ul>
<p><strong><code class="western">unknown_probes.json</code></strong><strong>
stays empty</strong></p>
<ul>
	<li><p>There may be no probe requests in the data, or everything is
	filtered as “known”.</p></li>
	<li><p>Run the monitor with <code class="western">--include-known
	--verbose</code> to confirm extraction is working.</p></li>
	<li><p>Verify that <code class="western">known_entities.json</code>
	isn’t overly broad (e.g., contains everything).</p></li>
</ul>
<p><strong>Different Kismet versions/exports</strong></p>
<ul>
	<li><p>The scripts already scan multiple JSON paths and, if needed,
	all tables for JSON blobs.</p></li>
	<li><p>If you see unexpected key names in your DB/XML, capture a
	small anonymized sample and adapt the field accessors similarly.</p></li>
</ul>
<hr/>

<p>Happy probing!</p>
<p style="line-height: 100%; margin-bottom: 0cm"><br/>

</p>
</body>
</html>
