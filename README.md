
  <h1>ESP32 CYD Stock & Crypto Ticker</h1>

  <p>
    This project turns an ESP32 2432S028R also called the Cheap Yellow Display or CYD into a live stock and crypto ticker for your desk. It shows real time prices on the built in touchscreen and is fully configurable through a simple web interface. No coding is required after flashing.
  </p>

  <h2>Features</h2>
  <ul>
    <li>Shows live prices for almost any cryptocurrency using CoinGecko and stocks using Finnhub</li>
    <li>Easy web based setup. Just connect, open your browser, and configure</li>
    <li>Automatic update intervals to avoid free API limits</li>
    <li>Displays backup data with an asterisk if APIs are temporarily unavailable</li>
    <li>Status LED for WiFi connection. You can turn it off in the settings</li>
    <li>Customizable. Track the symbols you care about</li>
  </ul>

  <h2>Hardware Requirements</h2>
  <ul>
    <li>ESP32 2432S028R Cheap Yellow Display board</li>
    <li>USB cable for flashing and power</li>
  </ul>

  <h2>Flashing the Firmware</h2>
  <p>
    You can use either the Arduino <code>.ino</code> source or the ready to flash <code>.bin</code> file from the GitHub repo. The easiest way is through your browser using <a href="https://web.esphome.io/" target="_blank">web.esphome.io</a>:
  </p>
  <ol>
    <li>Connect your ESP32 CYD board to your computer using USB</li>
    <li>Open <a href="https://web.esphome.io/" target="_blank">https://web.esphome.io/</a> in Chrome or Edge</li>
    <li>Click Connect and select your ESP32 device</li>
    <li>Choose the <code>.bin</code> firmware file you downloaded from GitHub</li>
    <li>Start the flashing process and wait until it finishes. The device will restart automatically</li>
  </ol>

  <h2>First Setup and Configuration</h2>
  <p>
    After flashing, the device will start its own WiFi access point called <code>StockTickerAP</code>. 
    <b>The default password is <code>password</code></b>
    Connect to this network with your phone or computer and open <code>http://192.168.4.1</code> in your browser.
  </p>
  <p>
    In the web interface you can
  </p>
  <ul>
    <li>Enter your home WiFi credentials</li>
    <li>Enter free API keys for CoinGecko and Finnhub</li>
    <li>Configure up to five cryptocurrencies and ten stock symbols</li>
    <li>Choose automatic or manual update intervals</li>
    <li>Enable or disable the status LED</li>
  </ul>

  <h2>How to Find the Right Symbols</h2>
  <h3>CoinGecko for Cryptocurrencies</h3>
  <ol>
    <li>Go to <a href="https://www.coingecko.com/en/coins/all" target="_blank">https://www.coingecko.com/en/coins/all</a></li>
    <li>Search for your cryptocurrency for example Solana Dogecoin or Cardano</li>
    <li>Click on the coin and check the URL. The last part is the CoinGecko ID for example <code>bitcoin</code> or <code>ethereum</code></li>
    <li>Use this ID in the ticker setup. Most common tickers like <code>BTC</code> or <code>ETH</code> are also supported and converted automatically</li>
  </ol>
  <h3>Finnhub for Stocks and ETFs</h3>
  <ol>
    <li>Go to <a href="https://stockanalysis.com/stocks/" target="_blank">https://stockanalysis.com/stocks/</a></li>
    <li>Search for your company or ticker for example Apple AAPL SAP or BMW</li>
    <li>The symbol is shown in the Symbol column for example <code>AAPL</code> for Apple <code>SAP</code> for SAP SE</li>
    <li>For German and many other international stocks you need to add the exchange suffix for example <code>SAP.DE</code> for SAP on the Frankfurt exchange</li>
    <li>Note that with the free Finnhub API you only get end of day data for international stocks like German tickers. Real time data is only available for US stocks in the free tier</li>
  </ol>
  <table>
    <tr>
      <th>Company</th>
      <th>Symbol for Ticker</th>
      <th>Data with Free API</th>
    </tr>
    <tr>
      <td>Apple</td>
      <td>AAPL</td>
      <td>Real time</td>
    </tr>
    <tr>
      <td>SAP</td>
      <td>SAP.DE</td>
      <td>End of day only</td>
    </tr>
    <tr>
      <td>BMW</td>
      <td>BMW.DE</td>
      <td>End of day only</td>
    </tr>
  </table>
  <p>
    If you are unsure just try your preferred symbol in the web setup. The system will check if it is valid and let you know if there is a problem via Serial Monitor.
  </p>

  <h2>Important Notes on CYD Variants</h2>
  <p>
    Not every Cheap Yellow Display board is exactly the same. There are small hardware differences between batches and sellers such as touch controller type and pin assignments. The firmware is tested on the most common ESP32 2432S028R variant. If your board uses a different touch controller or wiring you may need to adjust the pin definitions in the code. If the touchscreen or display does not work as expected check your board’s documentation or compare with the pin assignments in the repo.
  </p>

 
  <h2>License</h2>
  <p>
    This project is open source and free to use. Contributions and feedback are welcome
  </p>

</body>
</html>
