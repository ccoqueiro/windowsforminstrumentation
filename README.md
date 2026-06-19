# windowsforminstrumentation

# Guia de Instrumentação: OpenTelemetry em WinForms (.NET Framework 4.8)

Este guia orienta a integração do **OpenTelemetry (OTel)** em aplicações desktop baseadas em .NET Framework 4.8 Windows Forms. A configuração envia traces, métricas e eventos de ciclo de vida (workflows) diretamente para um **OpenTelemetry Collector**.

---

## 🚀 1. Instalação de Dependências

Abra o **Console do Gerenciador de Pacotes** no Visual Studio e instale os pacotes necessários:

```shell
Install-Package OpenTelemetry
Install-Package OpenTelemetry.Exporter.OpenTelemetryProtocol
Install-Package OpenTelemetry.Extensions.Hosting
```

---

## 🏛️ 2. Inicialização Global (`Program.cs`)

O ciclo de vida do OpenTelemetry deve iniciar no ponto de entrada do aplicativo (`Main`), antes de renderizar qualquer interface gráfica.

Substitua ou adapte o arquivo `Program.cs` conforme o modelo abaixo:

```csharp
using System;
using System.Diagnostics;
using System.Diagnostics.Metrics;
using System.Windows.Forms;
using OpenTelemetry;
using OpenTelemetry.Metrics;
using OpenTelemetry.Resources;
using OpenTelemetry.Trace;

namespace AplicacaoCliente
{
    static class Program
    {
        // Fontes globais de telemetria (Devem ser estáticas e reutilizadas)
        public static readonly ActivitySource WorkflowSource = new ActivitySource("Cliente.WinForms.Workflow");
        public static readonly Meter IndicadoresMeter = new Meter("Cliente.WinForms.Metrics");
        
        private static TracerProvider _tracerProvider;
        private static MeterProvider _meterProvider;

        [STAThread]
        static void Main()
        {
            Application.EnableVisualStyles();
            Application.SetCompatibleTextRenderingDefault(false);

            // Definição dos metadados da aplicação cliente
            var resourceBuilder = ResourceBuilder.CreateDefault()
                .AddService("NomeDaAplicacaoCliente", serviceVersion: "1.0.0");

            // Configuração do Coletor (Altere a URL conforme o ambiente do cliente)
            string collectorUri = "http://localhost:4317"; 

            // 1. Inicialização do Tracer (Traces e Eventos)
            _tracerProvider = Sdk.CreateTracerProviderBuilder()
                .SetResourceBuilder(resourceBuilder)
                .AddSource("Cliente.WinForms.Workflow")
                .AddOtlpExporter(opt => opt.Endpoint = new Uri(collectorUri))
                .Build();

            // 2. Inicialização do Meter (Métricas)
            _meterProvider = Sdk.CreateMeterProviderBuilder()
                .SetResourceBuilder(resourceBuilder)
                .AddMeter("Cliente.WinForms.Metrics")
                .AddOtlpExporter(opt => opt.Endpoint = new Uri(collectorUri))
                .Build();

            // Inicialização normal do WinForms
            Application.Run(new FormPrincipal());

            // Liberação de recursos e Flush pendente de dados ao fechar o App
            _tracerProvider?.Dispose();
            _meterProvider?.Dispose();
        }
    }
}
```

---

## 🛠️ 3. Rastreando Workflows e Eventos Customizados

Para identificar workflows de negócio e eventos de transição dentro da tela do usuário, utilize as propriedades `Activity` (Spans) e `ActivityEvent`.

### Exemplo de Implementação em Tela (`FormPrincipal.cs`):

```csharp
using System;
using System.Diagnostics;
using System.Diagnostics.Metrics;
using System.Collections.Generic;
using System.Threading.Tasks;
using System.Windows.Forms;

namespace AplicacaoCliente
{
    public partial class FormPrincipal : Form
    {
        // Contador customizado para volumetria de execuções
        private static readonly Counter<long> WorkflowCounter = 
            Program.IndicadoresMeter.CreateCounter<long>("workflows_processados_total");

        public FormPrincipal()
        {
            InitializeComponent();
        }

        private async void btnProcessar_Click(object sender, EventArgs e)
        {
            // Inicia o Bloco do Workflow (Rastreabilidade)
            using (Activity workflow = Program.WorkflowSource.StartActivity("WorkflowEmissaoNotaFiscal"))
            {
                // Tags estruturadas para filtros de busca no painel de observabilidade
                workflow?.SetTag("cliente.id", "99823");
                workflow?.SetTag("ambiente", "producao");

                try
                {
                    // --- EVENTO CUSTOMIZADO 1 ---
                    workflow?.AddEvent(new ActivityEvent("InicioValidacaoCadastral"));
                    await Task.Delay(800); // Simulação de processamento

                    // --- EVENTO CUSTOMIZADO 2 ---
                    workflow?.AddEvent(new ActivityEvent("FimValidacao_EnvioSefaz"));
                    await Task.Delay(1200); // Simulação de API externa

                    // Finaliza o Workflow sinalizando sucesso
                    workflow?.SetStatus(ActivityStatusCode.Ok);
                    
                    // Incrementa métrica de sucesso com atributos
                    WorkflowCounter.Add(1, new KeyValuePair<string, object>("status", "sucesso"));
                }
                catch (Exception ex)
                {
                    // Registra o erro detalhado no trace do coletor
                    workflow?.SetStatus(ActivityStatusCode.Error, ex.Message);
                    workflow?.RecordException(ex);
                    
                    // Incrementa métrica de falha com atributos
                    WorkflowCounter.Add(1, new KeyValuePair<string, object>("status", "falha"));
                    
                    MessageBox.Show("Erro ao processar fluxo interno.");
                }
            }
        }
    }
}
```

---

## 🗺️ 4. Conceitos Chave para o Cliente

Ao analisar os dados enviados, considere as seguintes equivalências conceituais do ecossistema .NET para o OpenTelemetry:

| Conceito de Negócio | Elemento Técnico .NET | Representação no Collector |
| :--- | :--- | :--- |
| **Ciclo/Workflow Completo** | `ActivitySource.StartActivity()` | **Span** (Início e Fim do processo com duração) |
| **Passo/Marcação de Etapa** | `ActivityEvent` | **Log Anotado** (Injetado dentro da linha do tempo do Span) |
| **Metadados de Negócio** | `SetTag("chave", "valor")` | **Attributes** (Campos indexados para busca e filtro) |
| **Gráficos e Alertas** | `Counter` / `Meter` | **Metrics** (Dados agregados de volumetria e erro) |

---

## 📌 Checklist de Validação

- [ ] Os pacotes NuGet estão na mesma versão majoritária para evitar conflitos de DLL no .NET 4.8.
- [ ] O nome do `ActivitySource` definido no `Program.cs` é **exatamente igual** ao adicionado em `.AddSource()`.
- [ ] O endpoint do OTel Collector (`http://localhost:4317`) está acessível a partir da rede onde a aplicação desktop roda.
- [ ] O OTel Collector do ambiente destino está configurado para aceitar recepção via protocolo **gRPC** (porta padrão `4317`).
