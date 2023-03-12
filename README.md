# TronNetCore
TronNetCore is a SDK that includes libraries for working with TRON, 
TronNetCore makes it easy to build TRON applications with .net.

## Get Started
### NuGet 

You can run the following command to install the `TronNetCore` in your project.

```
PM> Install-Package TronNetCore
```

### Configuration

First,You need to config `TronNetCore` in your `Startup.cs`:
```c#
......
using StowayNet;
using TronNet;
......

public void ConfigureServices(IServiceCollection services)
{
    ......

    services.AddTronNet(x =>
    {
        x.Network = TronNetwork.MainNet;
        x.Channel = new GrpcChannelOption { Host = "grpc.shasta.trongrid.io", Port = 50051 };
        x.SolidityChannel = new GrpcChannelOption { Host = "grpc.shasta.trongrid.io", Port = 50052 };
        x.ApiKey = "input your api key";
    });

    ......
}

```

### Sample

#### Sample 1: Generate Address Offline

```c#
using TronNet;

namespace TronNetTest
{
    class Class1
    {
        private readonly ITronClient _tronClient;

        public Class1(ITronClient tronClient)
        {
            _tronClient = tronClient;
        }

        public void GenerateAddress()
        {
            var key = TronECKey.GenerateKey(TronNetwork.MainNet);

            var address = key.GetPublicAddress();
        }
    }
}


```

#### Sample 2: Transaction Sign Offline
```c#
using TronNet;
using System.Threading.Tasks;
using Google.Protobuf;
using Microsoft.Extensions.Options;

namespace TronNetTest
{
    class Class1
    {
        private readonly ITransactionClient _transactionClient;
        private readonly IOptions<TronNetOptions> _options;
        public Class1(ITransactionClient transactionClient, IOptions<TronNetOptions> options)
        {
            _options = options;
            _transactionClient = transactionClient;
        }

        public async Task SignAsync()
        {
            var privateKey = "D95611A9AF2A2A45359106222ED1AFED48853D9A44DEFF8DC7913F5CBA727366";
            var ecKey = new TronECKey(privateKey, _options.Value.Network);
            var from = ecKey.GetPublicAddress();
            var to = "TGehVcNhud84JDCGrNHKVz9jEAVKUpbuiv";
            var amount = 100_000_000L;
            var transactionExtension = await _transactionClient.CreateTransactionAsync(from, to, amount);

            var transactionSigned = _transactionClient.GetTransactionSign(transactionExtension.Transaction, privateKey);
            
            var result = await _transactionClient.BroadcastTransactionAsync(transactionSigned);
        }
    }
}

```

#### Sample 3: Transfer TRX and TRC20 Transfer (USDT)
```c#
using TronNet.Accounts;
using TronNet.Contracts;
using TronNet;
using Microsoft.Extensions.DependencyInjection;
using Newtonsoft.Json;

namespace ConsoleApp1 {
    internal class Program {
        static async Task Main(string[] args) {
            Console.WriteLine("Program begin..");

            var privateKey = "D95611A9AF2A2A45359106222ED1AFED48853D9A44DEFF8DC7913F5CBA727366";

            //发送 trx
            var result = await TrxTransferAsync(privateKey, "TGehVcNhud84JDCGrNHKVz9jEAVKUpbuiv", 10000000L);
            Console.WriteLine(JsonConvert.SerializeObject(result));

            //发送 trc20 token usdt
            var transactionId = await EtherTransferAsync(privateKey, "TGehVcNhud84JDCGrNHKVz9jEAVKUpbuiv", 10, string.Empty);
            Console.WriteLine(transactionId);

            Console.WriteLine("Program end..\r\nPress any key to exit.");
            Console.ReadKey();
        }

        #region TrxTransferAsync

        private static async Task<dynamic> TrxTransferAsync(string privateKey, string to, long amount) {
            var record = TronServiceExtension.GetRecord();
            var transactionClient = record.TronClient?.GetTransaction();

            var account = new TronAccount(privateKey, TronNetwork.MainNet);

            var transactionExtension = await transactionClient?.CreateTransactionAsync(account.Address, to, amount)!;

            var transactionId = transactionExtension.Txid.ToStringUtf8();

            var transactionSigned = transactionClient.GetTransactionSign(transactionExtension.Transaction, privateKey);
            var returnObj = await transactionClient.BroadcastTransactionAsync(transactionSigned);

            return new { Result = returnObj.Result, Message = returnObj.Message, TransactionId = transactionId };
        }

        #endregion

        #region EtherTransferAsync

        private static async Task<string> EtherTransferAsync(string privateKey, string toAddress, decimal amount, string? memo) {
            const string contractAddress = "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t";

            var record = TronServiceExtension.GetRecord();
            var contractClientFactory = record.ServiceProvider.GetService<IContractClientFactory>();
            var contractClient = contractClientFactory?.CreateClient(ContractProtocol.TRC20);

            var account = new TronAccount(privateKey, TronNetwork.MainNet);

            const long feeAmount = 30 * 1000000L;

            return await contractClient.TransferAsync(contractAddress, account, toAddress, amount, memo, feeAmount);
        }

        #endregion
    }
}

```
