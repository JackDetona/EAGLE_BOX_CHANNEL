#region Using declarations
using System;
using System.Collections.Generic;
using System.ComponentModel;
using System.ComponentModel.DataAnnotations;
using System.Linq;
using System.Text;
using System.Threading;
using System.Threading.Tasks;
using System.Windows;
using System.Windows.Input;
using System.Windows.Media;
using System.Xml.Serialization;
using NinjaTrader.Cbi;
using NinjaTrader.Gui;
using NinjaTrader.Gui.Chart;
using NinjaTrader.Gui.SuperDom;
using NinjaTrader.Gui.Tools;
using NinjaTrader.Data;
using NinjaTrader.Core;
using NinjaTrader.NinjaScript.Indicators;
using NinjaTrader.NinjaScript.DrawingTools;
#endregion

namespace NinjaTrader.NinjaScript.Strategies
{
    public class EAGLE_BOX_CHANNEL : Strategy
    {
        private VOL vol;
        private SMA volMedia;
        private SMA tendenciaFiltro;
        private double pnlDiarioAcumulado;
        private bool travaDiariaAtivada;
        private int lastProcessedBar;
        private string ultimoStatusFiltro = "Eagle Filtro Rigoroso Ativo";

        private string atmId = "";
        private string orderId = "";
        private double pnlOperacaoAtual = 0.0;
        private bool operacaoFoiContabilizada = true;
        
        private DateTime dataInstalacao;
        private int ultimaDataSessao = -1;

        public enum OpcoesATM
        {
            [Description("DOW_JHONES_10")]
            DOW_JHONES_10,
            [Description("DOW_JHONES_20")]
            DOW_JHONES_20,
            [Description("DOW_JHONES_30")]
            DOW_JHONES_30,
            [Description("DOW_JHONES_5")]
            DOW_JHONES_5,
            [Description("OURO_2_CONTRATOS")]
            OURO_2_CONTRATOS,
            [Description("WS_100708")]
            WS_100708
        }

        protected override void OnStateChange()
        {
            if (State == State.SetDefaults)
            {
                Description = "EAGLE BOX CHANNEL - Operacional Filtrado e Blindado contra Falsos Sinais.";
                Name = "EAGLE_BOX_CHANNEL";
                Calculate = Calculate.OnEachTick; 
                EntriesPerDirection = 1;
                EntryHandling = EntryHandling.AllEntries;
                IsExitOnSessionCloseStrategy = true;
                ExitOnSessionCloseSeconds = 30;
                IsFillLimitOnTouch = false;

                SelecaoATM = OpcoesATM.WS_100708;
                
                LookbackCaixote = 20; // Período estrutural mais seguro para evitar ruídos
                CooldownCandles = 8;  // Espaçamento maior entre entradas para evitar excesso de trades

                MetaGainDiario = 750.0;
                LimiteLossDiario = 400.0;

                DiasDeValidade = 30;
                SenhaLicenca = "213211Jk@";
            }
            else if (State == State.Configure)
            {
                dataInstalacao = DateTime.Now;
            }
            else if (State == State.DataLoaded)
            {
                if (SenhaLicenca != "213211Jk@")
                {
                    Print("ERRO DE LICENÇA: Senha incorreta.");
                    Log("EAGLE BOX: Senha de licença inválida!", LogLevel.Error);
                    return;
                }

                vol = VOL();
                volMedia = SMA(vol, 20);
                tendenciaFiltro = SMA(Close, 50); // Filtro de tendência macro para proibir entradas contra o fluxo

                pnlDiarioAcumulado = 0.0;
                travaDiariaAtivada = false;
                lastProcessedBar = -1;
                pnlOperacaoAtual = 0.0;
                operacaoFoiContabilizada = true;
                ultimaDataSessao = -1;
            }
        }

        private string GetNomeAtmString()
        {
            switch (SelecaoATM)
            {
                case OpcoesATM.DOW_JHONES_10: return "DOW_JHONES_10";
                case OpcoesATM.DOW_JHONES_20: return "DOW_JHONES_20";
                case OpcoesATM.DOW_JHONES_30: return "DOW_JHONES_30";
                case OpcoesATM.DOW_JHONES_5: return "DOW_JHONES_5";
                case OpcoesATM.OURO_2_CONTRATOS: return "OURO_2_CONTRATOS";
                case OpcoesATM.WS_100708: return "WS_100708";
                default: return "WS_100708";
            }
        }

        protected override void OnBarUpdate()
        {
            if (SenhaLicenca != "213211Jk@" || DateTime.Now > dataInstalacao.AddDays(DiasDeValidade))
            {
                Draw.TextFixed(this, "ErroLicenca", "ERRO: LICENÇA EXPIRADA OU SENHA INVÁLIDA!", TextPosition.Center, Brushes.Red, new SimpleFont("Arial", 14), Brushes.Black, Brushes.Yellow, 100);
                return;
            }

            int dataAtualInt = ToDay(Time[0]);
            if (ultimaDataSessao != dataAtualInt)
            {
                ultimaDataSessao = dataAtualInt;
                pnlDiarioAcumulado = 0.0;
                travaDiariaAtivada = false;
                pnlOperacaoAtual = 0.0;
                operacaoFoiContabilizada = true;
            }

            AtualizarPnLDiario();
            DrawDashboard();

            if (travaDiariaAtivada) return;
            if (BarsInProgress != 0) return;

            if (CurrentBar < LookbackCaixote + 10) return;

            bool novoCandle = (CurrentBar > lastProcessedBar);
            bool estaFlatNoRobo = (Position.MarketPosition == MarketPosition.Flat) && string.IsNullOrEmpty(atmId);

            if (estaFlatNoRobo)
            {
                if (!novoCandle) return;
                if (CurrentBar < lastProcessedBar + CooldownCandles) return;

                // Mapeamento rigoroso do canal/caixote anterior (excluindo a barra atual)
                double maxCaixote = High[1];
                double minCaixote = Low[1];

                for (int i = 2; i <= LookbackCaixote; i++)
                {
                    if (High[i] > maxCaixote) maxCaixote = High[i];
                    if (Low[i] < minCaixote) minCaixote = Low[i];
                }

                // Desenha o canal estrutural limpo na tela
                Draw.Rectangle(this, "CanalRigoroso_" + CurrentBar, false, Time[LookbackCaixote], maxCaixote, Time[0], minCaixote, Brushes.Transparent, Brushes.DodgerBlue, 2);

                // Critérios estritos exigidos pelo operacional
                bool volumeForte = vol[0] >= (volMedia[0] * 0.70); // Exige volume expressivo para evitar rompimento falso
                double corpoCandle = Math.Abs(Close[0] - Open[0]);
                bool candleExpansao = corpoCandle >= (TickSize * 4); // Candle de expansão real

                // Filtro de direção: Só compra se estiver acima da média de tendência e só vende se estiver abaixo
                bool aFavorDaTendenciaAlta = Close[0] > tendenciaFiltro[0];
                bool aFavorDaTendenciaBaixa = Close[0] < tendenciaFiltro[0];

                bool rompeuTopoComCorpo = (Close[0] > maxCaixote) && (Close[0] > Open[0]);
                bool rompeuFundoComCorpo = (Close[0] < minCaixote) && (Close[0] < Open[0]);

                bool executarCompra = volumeForte && candleExpansao && rompeuTopoComCorpo && aFavorDaTendenciaAlta;
                bool executarVenda  = volumeForte && candleExpansao && rompeuFundoComCorpo && aFavorDaTendenciaBaixa;

                if (executarCompra)
                {
                    lastProcessedBar = CurrentBar;
                    DispararCompraATM();
                    
                    Draw.ArrowUp(this, "BuyClean_" + CurrentBar, true, 0, Low[0] - (TickSize * 8), Brushes.Lime);
                    TocarAlarmeCashRegister();
                    
                    ultimoStatusFiltro = "COMPRA VÁLIDA EXECUTADA";
                    return;
                }

                if (executarVenda)
                {
                    lastProcessedBar = CurrentBar;
                    DispararVendaATM();
                    
                    Draw.ArrowDown(this, "SellClean_" + CurrentBar, true, 0, High[0] + (TickSize * 8), Brushes.Red);
                    TocarAlarmeCashRegister();
                    
                    ultimoStatusFiltro = "VENDA VÁLIDA EXECUTADA";
                    return;
                }

                ultimoStatusFiltro = "Aguardando Rompimento Válido com Filtro";
            }
        }

        private void TocarAlarmeCashRegister()
        {
            Task.Run(() =>
            {
                try
                {
                    int ciclos = 6; 
                    for (int i = 0; i < ciclos; i++)
                    {
                        Application.Current.Dispatcher.Invoke(() =>
                        {
                            NinjaTrader.Core.Globals.PlaySound(NinjaTrader.Core.Globals.InstallDir + "\\sounds\\CashRegister.wav");
                        });
                        Thread.Sleep(1200);
                    }
                }
                catch { }
            });
        }

        private void DispararCompraATM()
        {
            try
            {
                atmId = "ATM_CleanBuy_" + CurrentBar + "_" + DateTime.Now.Ticks;
                orderId = "Order_CleanBuy_" + CurrentBar + "_" + DateTime.Now.Ticks;
                operacaoFoiContabilizada = false;

                AtmStrategyCreate(OrderAction.Buy, OrderType.Market, 0, 0, TimeInForce.Day, orderId, GetNomeAtmString(), atmId, (errorCode, id) =>
                {
                    if (errorCode != ErrorCode.NoError)
                    {
                        ultimoStatusFiltro = "ERRO ATM COMPRA: " + errorCode;
                        atmId = "";
                    }
                });
            }
            catch (Exception ex)
            {
                ultimoStatusFiltro = "EXCEÇÃO COMPRA: " + ex.Message;
                atmId = "";
            }
        }

        private void DispararVendaATM()
        {
            try
            {
                atmId = "ATM_CleanSell_" + CurrentBar + "_" + DateTime.Now.Ticks;
                orderId = "Order_CleanSell_" + CurrentBar + "_" + DateTime.Now.Ticks;
                operacaoFoiContabilizada = false;

                AtmStrategyCreate(OrderAction.SellShort, OrderType.Market, 0, 0, TimeInForce.Day, orderId, GetNomeAtmString(), atmId, (errorCode, id) =>
                {
                    if (errorCode != ErrorCode.NoError)
                    {
                        ultimoStatusFiltro = "ERRO ATM VENDA: " + errorCode;
                        atmId = "";
                    }
                });
            }
            catch (Exception ex)
            {
                ultimoStatusFiltro = "EXCEÇÃO VENDA: " + ex.Message;
                atmId = "";
            }
        }

        private void AtualizarPnLDiario()
        {
            if (!string.IsNullOrEmpty(atmId))
            {
                try
                {
                    MarketPosition posAtm = GetAtmStrategyMarketPosition(atmId);
                    if (posAtm != MarketPosition.Flat)
                    {
                        double pnlRealizado = GetAtmStrategyRealizedProfitLoss(atmId);
                        double pnlAberto = GetAtmStrategyUnrealizedProfitLoss(atmId);
                        pnlOperacaoAtual = pnlRealizado + pnlAberto;

                        double pnlTotalEstimado = pnlDiarioAcumulado + pnlOperacaoAtual;

                        if (!travaDiariaAtivada && (pnlTotalEstimado >= MetaGainDiario || pnlTotalEstimado <= -LimiteLossDiario))
                        {
                            AtmStrategyClose(atmId);
                            travaDiariaAtivada = true;
                            ultimoStatusFiltro = pnlTotalEstimado >= MetaGainDiario ? "TRAVADO: META GAIN ATINGIDA!" : "TRAVADO: STOP LOSS ($400)!";
                        }
                    }
                    else
                    {
                        if (!operacaoFoiContabilizada)
                        {
                            double pnlFechado = GetAtmStrategyRealizedProfitLoss(atmId);
                            pnlDiarioAcumulado += pnlFechado;
                            operacaoFoiContabilizada = true;
                        }

                        pnlOperacaoAtual = 0.0;

                        if (!travaDiariaAtivada && (pnlDiarioAcumulado >= MetaGainDiario || pnlDiarioAcumulado <= -LimiteLossDiario))
                        {
                            travaDiariaAtivada = true;
                            ultimoStatusFiltro = pnlDiarioAcumulado >= MetaGainDiario ? "TRAVADO: META GAIN ATINGIDA!" : "TRAVADO: STOP LOSS ($400)!";
                        }

                        atmId = ""; 
                    }
                }
                catch 
                { 
                    pnlOperacaoAtual = 0.0;
                }
            }
            else
            {
                pnlOperacaoAtual = 0.0;
                if (!travaDiariaAtivada && (pnlDiarioAcumulado >= MetaGainDiario || pnlDiarioAcumulado <= -LimiteLossDiario))
                {
                    travaDiariaAtivada = true;
                }
            }
        }

        private void DrawDashboard()
        {
            string statusTexto = "ROBÔ FILTRADO E BLINDADO";

            if (travaDiariaAtivada)
            {
                statusTexto = pnlDiarioAcumulado >= MetaGainDiario ? "BLOQUEADO: META GAIN!" : "BLOQUEADO: STOP LOSS ($400)!";
            }

            string statusOperacaoTexto = "FLAT (Sem Posição)";
            if (!string.IsNullOrEmpty(atmId))
            {
                try
                {
                    MarketPosition pos = GetAtmStrategyMarketPosition(atmId);
                    if (pos == MarketPosition.Long)
                        statusOperacaoTexto = string.Format("COMPRADO | PnL Aberto: ${0:F2}", pnlOperacaoAtual);
                    else if (pos == MarketPosition.Short)
                        statusOperacaoTexto = string.Format("VENDIDO | PnL Aberto: ${0:F2}", pnlOperacaoAtual);
                }
                catch
                {
                    statusOperacaoTexto = "EM OPERAÇÃO";
                }
            }

            double pnlTotalExibicao = pnlDiarioAcumulado + pnlOperacaoAtual;

            string painelTexto = string.Format(
                "=== EAGLE CLEAN STRATEGY ===\n" +
                "Status: {0}\n" +
                "Posição: {1}\n" +
                "Diagnóstico: {2}\n" +
                "ATM Ativa: {3}\n" +
                "PnL Total (Dia): ${4:F2}\n" +
                "-----------------------------------------\n" +
                "Meta Diária: ${5:F2} | Stop Diário: ${6:F2}",
                statusTexto,
                statusOperacaoTexto,
                ultimoStatusFiltro,
                GetNomeAtmString(),
                pnlTotalExibicao,
                MetaGainDiario,
                LimiteLossDiario
            );

            Draw.TextFixed(this, "BoxDashboard", painelTexto, TextPosition.TopLeft, Brushes.White, new SimpleFont("Arial", 11), Brushes.Black, Brushes.DimGray, 80);
        }

        #region Properties
        [NinjaScriptProperty]
        [XmlIgnore]
        [Display(Name="Selecione a ATM", Description="Escolha a estratégia ATM", Order=1, GroupName="ATM")]
        public OpcoesATM SelecaoATM { get; set; }

        [Browsable(false)]
        public int SelecaoATMSerialization
        {
            get { return (int)SelecaoATM; }
            set { SelecaoATM = (OpcoesATM)value; }
        }

        [NinjaScriptProperty]
        [Range(5, 50)]
        [Display(Name="Lookback Canal", Description="Período do canal estrutural", Order=2, GroupName="Parâmetros")]
        public int LookbackCaixote { get; set; }

        [NinjaScriptProperty]
        [Range(1, 30)]
        [Display(Name="Cooldown Candles", Description="Espaçamento entre entradas", Order=3, GroupName="Parâmetros")]
        public int CooldownCandles { get; set; }

        [NinjaScriptProperty]
        [Display(Name="Meta Gain ($)", Description="Meta de Gain Diário", Order=4, GroupName="Risco")]
        public double MetaGainDiario { get; set; }

        [NinjaScriptProperty]
        [Display(Name="Limite Loss ($)", Description="Limite de Loss Diário ($400)", Order=5, GroupName="Risco")]
        public double LimiteLossDiario { get; set; }

        [NinjaScriptProperty]
        [Range(1, 365)]
        [Display(Name="Dias de Validade", Description="Validade", Order=6, GroupName="Licença")]
        public int DiasDeValidade { get; set; }

        [NinjaScriptProperty]
        [Display(Name="Senha de Licença", Description="Senha de acesso", Order=7, GroupName="Licença")]
        public string SenhaLicenca { get; set; }
        #endregion
    }
}
