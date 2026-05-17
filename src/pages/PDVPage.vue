<template>
  <q-page class="bg-grey-2 q-pa-md">
    <div class="row q-col-gutter-md items-start">
      <div class="col-12 col-md-8">
        <q-card flat bordered class="q-mb-md border">
          <q-card-section>
            <div class="text-h5">Terminal Punto de Venta</div>
            <div class="text-caption text-grey-7">
              Venta rápida con selección de cliente y control automático de estoque
            </div>
          </q-card-section>
        </q-card>

        <q-card flat bordered class="q-mb-md border">
          <q-card-section>
            <div class="text-subtitle1 text-weight-medium q-mb-md">Cliente de la Venta</div>

            <q-select
              v-model="clienteSelecionado"
              class="border"
              :options="clientesOptions"
              option-label="label"
              option-value="value"
              emit-value
              map-options
              outlined
              dense
              clearable
              use-input
              input-debounce="300"
              label="Seleccionar cliente (opcional)"
              @filter="filtrarClientes"
            />
          </q-card-section>
        </q-card>

        <q-card flat bordered class="border">
          <q-card-section>
            <div class="text-subtitle1 text-weight-medium q-mb-md">Productos</div>

            <q-input
              ref="inputBusca"
              v-model="busca"
              outlined
              dense
              label="Buscar producto, ID o código de barras"
              @keyup.enter="buscarOuAdicionarProduto"
            >
              <template #prepend>
                <q-icon name="search" />
              </template>
            </q-input>

            <div class="lista-produtos">
              <q-list bordered separator>
                <q-item
                  v-for="produto in produtosFiltrados"
                  :key="produto.id"
                  clickable
                  dense
                  @click="adicionarProduto(produto)"
                >
                  <q-item-section>
                    <q-item-label>{{ produto.nome }}</q-item-label>
                    <q-item-label caption> Estoque: {{ produto.estoque }} </q-item-label>
                  </q-item-section>

                  <q-item-section side>
                    <div class="text-weight-medium">
                      {{ formatarMoeda(produto.preco) }}
                    </div>
                  </q-item-section>

                  <q-item-section side>
                    <q-btn
                      flat
                      round
                      color="primary"
                      icon="add_shopping_cart"
                      @click.stop="adicionarProduto(produto)"
                    />
                  </q-item-section>
                </q-item>
              </q-list>
            </div>
          </q-card-section>
        </q-card>
      </div>

      <div class="col-12 col-md-4">
        <q-card flat bordered class="sticky-card border">
          <q-card-section>
            <div class="text-h6 q-mb-md">Carrinho</div>

            <div class="q-mb-md">
              <div class="text-caption text-grey-7">Cliente final</div>
              <div class="text-subtitle2">
                {{ nomeClienteFinal }}
              </div>
            </div>

            <q-separator class="q-my-md" />

            <div v-if="carrinho.length === 0" class="text-grey-7 text-center q-py-xl">
              Ningún ítem adicionado
            </div>

            <div v-else>
              <div ref="scrollCarrinhoRef" class="scroll-carrinho">
                <div
                  v-for="item in carrinho"
                  :key="item.produto_id"
                  class="q-mb-md q-pa-sm rounded-borders bg-grey-1"
                >
                  <div class="row items-center justify-between q-col-gutter-sm">
                    <div class="col-auto">
                      <q-avatar size="55px" rounded>
                        <img v-if="item.foto" :src="item.foto" alt="Imagen del producto" />
                        <q-icon v-else name="image" color="grey-5" />
                      </q-avatar>
                    </div>
                    <div class="col" style="margin-top: -15px">
                      <div class="text-weight-medium">{{ item.nome }}</div>
                      <div class="text-caption text-grey-7">
                        {{ formatarMoeda(item.preco) }} por unidad
                      </div>
                    </div>

                    <div class="col-auto">
                      <q-btn
                        flat
                        round
                        dense
                        icon="remove"
                        @click="diminuirQuantidade(item.produto_id)"
                      />
                    </div>

                    <div class="col-auto">
                      <div class="text-subtitle2">{{ item.quantidade }}</div>
                    </div>

                    <div class="col-auto">
                      <q-btn
                        flat
                        round
                        dense
                        icon="add"
                        @click="aumentarQuantidade(item.produto_id)"
                      />
                    </div>

                    <div class="col-auto">
                      <q-btn
                        flat
                        round
                        dense
                        color="negative"
                        icon="delete"
                        @click="removerItem(item.produto_id)"
                      />
                    </div>
                  </div>

                  <div class="row justify-between q-mt-sm">
                    <span class="text-grey-7">Subtotal</span>
                    <strong>{{ formatarMoeda(item.subtotal) }}</strong>
                  </div>
                </div>
              </div>
            </div>

            <q-separator class="q-my-md" />

            <div class="row justify-between q-mb-sm">
              <span class="text-grey-7">Itens</span>
              <strong>{{ totalItens }}</strong>
            </div>

            <div class="row q-col-gutter-sm q-mb-md">
              <div class="col-12 col-sm-6">
                <q-input
                  v-model.number="descontoValor"
                  type="number"
                  min="0"
                  :max="descontoTipo === 'percentual' ? 100 : undefined"
                  outlined
                  label="Descuento"
                  dense
                >
                  <template #prepend>
                    <q-btn-toggle
                      v-model="descontoTipo"
                      no-caps
                      unelevated
                      dense
                      toggle-color="primary"
                      color="white"
                      text-color="primary"
                      :options="[
                        { label: 'R$', value: 'valor' },
                        { label: '%', value: 'percentual' },
                      ]"
                    />
                  </template>
                </q-input>
              </div>

              <div class="col-12 col-sm-6">
                <q-input
                  v-model.number="acrescimoValor"
                  type="number"
                  min="0"
                  :max="acrescimoTipo === 'percentual' ? 100 : undefined"
                  outlined
                  label="Recargo"
                  dense
                >
                  <template #prepend>
                    <q-btn-toggle
                      v-model="acrescimoTipo"
                      no-caps
                      unelevated
                      dense
                      toggle-color="primary"
                      color="white"
                      text-color="primary"
                      :options="[
                        { label: 'R$', value: 'valor' },
                        { label: '%', value: 'percentual' },
                      ]"
                    />
                  </template>
                </q-input>
              </div>
            </div>

            <div class="row justify-between q-mb-sm">
              <span class="text-grey-7">Subtotal</span>
              <strong>{{ formatarMoeda(subtotalVenda) }}</strong>
            </div>

            <div class="row justify-between q-mb-sm">
              <span class="text-grey-7 subir">
                Descuento
                <small v-if="descontoTipo === 'percentual'"> ({{ descontoValor }}%) </small>
              </span>
              <strong class="subir">{{ formatarMoeda(descontoCalculado) }}</strong>
            </div>

            <div class="row justify-between q-mb-sm subir">
              <span class="text-grey-7">
                Recargo
                <small v-if="acrescimoTipo === 'percentual'"> ({{ acrescimoValor }}%) </small>
              </span>
              <strong>{{ formatarMoeda(acrescimoCalculado) }}</strong>
            </div>

            <div class="row justify-between q-mb-sm subir">
              <span class="text-grey-7">Base imponible</span>
              <strong>{{ formatarMoeda(baseImponible) }}</strong>
            </div>

            <div class="row justify-between q-mb-sm subir">
              <span class="text-grey-7">Exento / Exonerado</span>
              <strong>{{ formatarMoeda(subtotalExentoAjustado) }}</strong>
            </div>

            <div class="row justify-between q-mb-sm subir">
              <span class="text-grey-7">IVA 16%</span>
              <strong>{{ formatarMoeda(ivaCalculado) }}</strong>
            </div>

            <div class="row justify-between q-mb-md">
              <span class="text-grey-7">Total</span>
              <strong class="text-primary text-h5">
                {{ formatarMoeda(totalVenda) }}
              </strong>
            </div>

            <q-separator class="q-my-md" />

            <q-btn
              color="positive"
              icon="point_of_sale"
              label="Faturar"
              class="full-width"
              :disable="carrinho.length === 0"
              @click="modalFaturar = true"
            />

            <!-- FORMA DE PAGAMENTO -->
            <q-dialog v-model="modalFaturar" persistent>
              <q-card class="border" style="width: 450px">
                <q-card-section>
                  <div class="text-h6">Facturamiento</div>
                </q-card-section>

                <q-card-section>
                  <div
                    v-for="(pagamento, index) in pagamentos"
                    :key="pagamento.forma"
                    class="row items-center q-col-gutter-sm q-mb-sm"
                  >
                    <div class="col-4">
                      <div class="text-subtitle2">
                        {{ pagamento.label }}
                      </div>
                    </div>

                    <div class="col-8">
                      <q-input
                        :ref="criarPagamentoRef(index)"
                        v-model.number="pagamento.valor"
                        type="number"
                        min="0"
                        step="0.01"
                        outlined
                        dense
                        class="sem-setas"
                        input-class="text-right"
                        placeholder="0,00"
                        :label="`Valor en ${pagamento.label}`"
                        :disable="
                          Number(totalVenda || 0) > 0 &&
                          pagamentos.some(
                            (outroPagamento, outroIndex) =>
                              outroIndex !== index &&
                              round2(Number(outroPagamento.valor || 0)) >=
                                round2(Number(totalVenda || 0)),
                          )
                        "
                        @update:model-value="
                          () => {
                            onPagamentoChange(index);

                            if (
                              Number(totalVenda || 0) > 0 &&
                              round2(Number(pagamento.valor || 0)) >=
                                round2(Number(totalVenda || 0))
                            ) {
                              pagamentos.forEach((outroPagamento, outroIndex) => {
                                if (outroIndex !== index) {
                                  outroPagamento.valor = 0;
                                }
                              });
                            }
                          }
                        "
                      >
                        <template #prepend>
                          <span class="text-blue-7 text-caption">R$</span>
                        </template>
                      </q-input>
                    </div>
                  </div>

                  <q-separator class="q-my-md" />

                  <div class="row justify-between q-mb-sm">
                    <span class="text-grey-7">Base imponible</span>
                    <strong>{{ formatarMoeda(baseImponible) }}</strong>
                  </div>

                  <div class="row justify-between q-mb-sm">
                    <span class="text-grey-7">Exento / Exonerado</span>
                    <strong>{{ formatarMoeda(subtotalExentoAjustado) }}</strong>
                  </div>

                  <div class="row justify-between q-mb-sm">
                    <span class="text-grey-7">IVA 16%</span>
                    <strong>{{ formatarMoeda(ivaCalculado) }}</strong>
                  </div>

                  <div class="row justify-between q-mb-sm">
                    <span class="text-grey-7">Total</span>
                    <strong class="text-primary text-h6">
                      {{ formatarMoeda(totalVenda) }}
                    </strong>
                  </div>

                  <div class="row justify-between q-mb-sm">
                    <span class="text-grey-7">Total pago</span>
                    <strong>{{ formatarMoeda(totalPago) }}</strong>
                  </div>

                  <div class="row justify-between q-mb-sm">
                    <span class="text-grey-7">Falta pagar</span>
                    <strong class="text-negative">
                      {{ formatarMoeda(faltaPagar) }}
                    </strong>
                  </div>

                  <div class="row justify-between q-mb-md">
                    <span class="text-grey-7">Cambio</span>
                    <strong class="text-positive text-h6">
                      {{ formatarMoeda(troco) }}
                    </strong>
                  </div>

                  <q-checkbox
                    v-model="imprimirComprovanteAutomaticamente"
                    dense
                    label="Imprimir comprobante después de concluir"
                  />
                </q-card-section>

                <q-card-actions align="right">
                  <q-btn flat label="Cancelar" v-close-popup />
                  <q-btn
                    color="positive"
                    icon="check"
                    label="Confirmar"
                    :loading="finalizando"
                    :disable="!validartarjeta"
                    @click="finalizarVenda"
                  />
                </q-card-actions>
              </q-card>
            </q-dialog>
          </q-card-section>
        </q-card>
      </div>
    </div>
  </q-page>
</template>

<script setup lang="ts">
import { computed, nextTick, onMounted, ref, watch, type ComponentPublicInstance } from 'vue';
import { Notify, QInput } from 'quasar';
import { api } from 'boot/axios';
import axios from 'axios';

const API_URL = 'http://localhost:3000';

type TipoAjuste = 'valor' | 'percentual';
type TipoImposto = 'GRAVADO' | 'EXENTO' | 'EXONERADO';
type FormaPagamento = 'EFECTIVO' | 'TARJETA' | 'PAGOMOVIL';
type FormaPagamentoResumo = FormaPagamento | 'COMBINADO';

interface Cliente {
  id: number;
  nome: string;
  email: string;
  telefone?: string;
  cidade?: string;
  status?: string;
}

interface Produto {
  id: number;
  nome: string;
  categoria?: string;
  preco: number;
  estoque: number;
  status?: string;
  foto: string;
  codigo_barras?: string;
  tipo_imposto?: TipoImposto;
  aliquota_iva?: number;
}

interface OptionItem {
  label: string;
  value: number;
}

interface ItemCarrinho {
  produto_id: number;
  nome: string;
  preco: number;
  quantidade: number;
  subtotal: number;
  estoqueDisponivel: number;
  foto: string;
  tipo_imposto: TipoImposto;
  aliquota_iva: number;
}

interface PagamentoInformado {
  forma: FormaPagamento;
  valor: number;
}

interface DadosComprovanteVenda {
  pedidoId?: number | string;
  dataVenda: Date;
  clienteNome: string;
  itens: ItemCarrinho[];
  pagamentos: PagamentoInformado[];
  subtotal: number;
  desconto: number;
  acrescimo: number;
  baseImponible: number;
  subtotalExento: number;
  iva: number;
  total: number;
  totalPago: number;
  troco: number;
}

const busca = ref('');
const validartarjeta = ref(true);
const finalizando = ref(false);
const inputBusca = ref();

const clientes = ref<Cliente[]>([]);
const produtos = ref<Produto[]>([]);

const clientesOptions = ref<OptionItem[]>([]);
const clienteSelecionado = ref<number | null>(null);

const carrinho = ref<ItemCarrinho[]>([]);

const descontoTipo = ref<TipoAjuste>('valor');
const descontoValor = ref<number>(0);

const acrescimoTipo = ref<TipoAjuste>('valor');
const acrescimoValor = ref<number>(0);

const CLIENTE_PADRAO_NOME = 'Consumidor Final';

const clientePadrao = computed(
  () => clientes.value.find((cliente) => cliente.nome === CLIENTE_PADRAO_NOME) || null,
);

const clienteFinalId = computed(() => clienteSelecionado.value || clientePadrao.value?.id || null);

const beep = new Audio('/src/assets/beep-02.mp3');
const beepErro = new Audio('/src/assets/error-sounds.mp3');

const LABEL_FORMA_PAGAMENTO: Record<FormaPagamento, string> = {
  EFECTIVO: 'Efectivo',
  TARJETA: 'Tarjeta',
  PAGOMOVIL: 'Pago Movil',
};

const pagamentos = ref<{ forma: FormaPagamento; label: string; valor: number | null }[]>([
  { forma: 'EFECTIVO', label: 'Efectivo', valor: null },
  { forma: 'TARJETA', label: 'Tarjeta', valor: null },
  { forma: 'PAGOMOVIL', label: 'Pago Movil', valor: null },
]);

const modalFaturar = ref(false);
const imprimirComprovanteAutomaticamente = ref(true);

const pagamentoRefs = ref<Array<InstanceType<typeof QInput> | null>>([]);

function criarPagamentoRef(index: number) {
  return (el: Element | ComponentPublicInstance | null) => {
    pagamentoRefs.value[index] = el as InstanceType<typeof QInput> | null;
  };
}

watch(modalFaturar, async (abriu) => {
  if (!abriu) {
    pagamentoRefs.value = [];
    return;
  }

  await nextTick();

  setTimeout(() => {
    pagamentoRefs.value[0]?.focus();
  }, 200);
});

const round2 = (value: number) => Math.round((value + Number.EPSILON) * 100) / 100;

const isFormaSemTroco = (forma?: string) =>
  ['PAGOMOVIL', 'TARJETA'].includes((forma || '').toUpperCase());

const totalInformado = computed(() => {
  const total = pagamentos.value.reduce((acc, item) => {
    return acc + (Number(item.valor) || 0);
  }, 0);

  return round2(total);
});

const totalEmDinheiro = computed(() => {
  const totalDinheiroInformado = pagamentos.value
    .filter((item) => item.forma === 'EFECTIVO')
    .reduce((acc, item) => acc + (Number(item.valor) || 0), 0);

  return round2(Math.max(0, totalDinheiroInformado - troco.value));
});

const totalPago = computed(() => {
  return round2(Math.min(totalInformado.value, totalVenda.value));
});

const faltaPagar = computed(() => {
  return round2(Math.max(0, totalVenda.value - totalInformado.value));
});

const troco = computed(() => {
  return round2(Math.max(0, totalInformado.value - totalVenda.value));
});

function tocarBeep() {
  beep.currentTime = 0;
  beep.play();
}

const nomeClienteFinal = computed(() => {
  if (clienteSelecionado.value) {
    const cliente = clientes.value.find((c) => c.id === clienteSelecionado.value);
    return cliente?.nome || CLIENTE_PADRAO_NOME;
  }

  return clientePadrao.value?.nome || CLIENTE_PADRAO_NOME;
});

const produtosFiltrados = computed(() => {
  const termo = busca.value.toLowerCase().trim();

  return produtos.value.filter((produto) => {
    const ativo = produto.status !== 'INACTIVO';

    if (!termo) {
      return ativo;
    }

    const buscaPorNome = produto.nome.toLowerCase().includes(termo);
    const buscaPorId = String(produto.id).includes(termo);
    const buscaPorCodigoBarras = String(produto.codigo_barras || '').includes(termo);

    return ativo && (buscaPorNome || buscaPorId || buscaPorCodigoBarras);
  });
});

const totalItens = computed(() => carrinho.value.reduce((acc, item) => acc + item.quantidade, 0));

const subtotalVenda = computed(() => carrinho.value.reduce((acc, item) => acc + item.subtotal, 0));

const descontoCalculado = computed(() => {
  if (descontoTipo.value === 'percentual') {
    return subtotalVenda.value * (Number(descontoValor.value || 0) / 100);
  }

  return Number(descontoValor.value || 0);
});

const acrescimoCalculado = computed(() => {
  if (acrescimoTipo.value === 'percentual') {
    return subtotalVenda.value * (Number(acrescimoValor.value || 0) / 100);
  }

  return Number(acrescimoValor.value || 0);
});

function itemGravadoComIva(item: ItemCarrinho): boolean {
  return item.tipo_imposto === 'GRAVADO' && Number(item.aliquota_iva || 0) > 0;
}

const subtotalGravadoVenda = computed(() => {
  return round2(
    carrinho.value
      .filter((item) => itemGravadoComIva(item))
      .reduce((acc, item) => acc + item.subtotal, 0),
  );
});

const subtotalExentoVenda = computed(() => {
  return round2(
    carrinho.value
      .filter((item) => !itemGravadoComIva(item))
      .reduce((acc, item) => acc + item.subtotal, 0),
  );
});

const totalAntesImposto = computed(() => {
  return round2(
    Math.max(0, subtotalVenda.value - descontoCalculado.value + acrescimoCalculado.value),
  );
});

const fatorAjusteVenda = computed(() => {
  if (subtotalVenda.value <= 0) return 0;

  return totalAntesImposto.value / subtotalVenda.value;
});

const baseImponible = computed(() => {
  return round2(subtotalGravadoVenda.value * fatorAjusteVenda.value);
});

const subtotalExentoAjustado = computed(() => {
  return round2(subtotalExentoVenda.value * fatorAjusteVenda.value);
});

const ivaCalculado = computed(() => {
  const totalIva = carrinho.value.reduce((acc, item) => {
    if (!itemGravadoComIva(item)) return acc;

    const baseItemAjustada = item.subtotal * fatorAjusteVenda.value;
    const aliquota = Number(item.aliquota_iva || 0) / 100;

    return acc + baseItemAjustada * aliquota;
  }, 0);

  return round2(totalIva);
});

const totalVenda = computed(() => {
  return round2(totalAntesImposto.value + ivaCalculado.value);
});

function formatarMoeda(valor: number): string {
  return new Intl.NumberFormat('pt-BR', {
    style: 'currency',
    currency: 'BRL',
  }).format(Number(valor || 0));
}

function formatarDataHora(data = new Date()): string {
  return new Intl.DateTimeFormat('pt-BR', {
    dateStyle: 'short',
    timeStyle: 'medium',
  }).format(data);
}

function escapeHtml(valor: string | number | null | undefined): string {
  return String(valor ?? '')
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#39;');
}

function obterResumoFormaPagamento(
  pagamentosPayload: Array<{ forma: FormaPagamento; valor: number }>,
): FormaPagamentoResumo {
  const formasValidas = Array.from(
    new Set(
      pagamentosPayload.filter((item) => Number(item.valor || 0) > 0).map((item) => item.forma),
    ),
  );

  const [formaUnica] = formasValidas;

  if (formasValidas.length === 1 && formaUnica) {
    return formaUnica;
  }

  return 'COMBINADO';
}

function gerarHtmlComprovanteVenda(dados: DadosComprovanteVenda): string {
  const linhasItens = dados.itens
    .map(
      (item) => `
        <div class="linha">
          <span>${escapeHtml(item.nome)} x${item.quantidade}</span>
          <span>${formatarMoeda(item.subtotal)}</span>
        </div>
        <div class="linha detalhe">
          <span>${formatarMoeda(item.preco)} por un.</span>
          <span></span>
        </div>
      `,
    )
    .join('');

  const linhasPagamentos = dados.pagamentos
    .map(
      (pagamento) => `
        <div class="linha">
          <span>${escapeHtml(LABEL_FORMA_PAGAMENTO[pagamento.forma])}</span>
          <span>${formatarMoeda(pagamento.valor)}</span>
        </div>
      `,
    )
    .join('');

  return `
    <!DOCTYPE html>
    <html lang="pt-BR">
      <head>
        <meta charset="UTF-8" />
        <title>Comprobante de venta</title>
        <style>
          @page {
            size: 80mm auto;
            margin: 4mm;
          }

          * {
            box-sizing: border-box;
          }

          body {
            margin: 0;
            padding: 0;
            color: #000;
            background: #fff;
            font-family: monospace;
            font-size: 12px;
          }

          .comprovante {
            width: 72mm;
            margin: 0 auto;
          }

          .centro {
            text-align: center;
          }

          .titulo {
            font-size: 15px;
            font-weight: 700;
            margin-bottom: 2px;
          }

          .subtitulo {
            font-size: 11px;
            margin-bottom: 8px;
          }

          .linha {
            display: flex;
            justify-content: space-between;
            align-items: flex-start;
            gap: 8px;
            margin: 2px 0;
            white-space: pre-wrap;
            word-break: break-word;
          }

          .detalhe {
            font-size: 11px;
            color: #444;
          }

          .separador {
            border-top: 1px dashed #000;
            margin: 8px 0;
          }

          .total {
            font-size: 14px;
            font-weight: 700;
          }
        </style>
      </head>

      <body>
        <div class="comprovante">
          <div class="centro">
            <div class="titulo">COMPROBANTE DE VENTA</div>
            <div class="subtitulo">Emitido por TPV</div>
          </div>

          <div class="linha">
            <span>Fecha</span>
            <span>${escapeHtml(formatarDataHora(dados.dataVenda))}</span>
          </div>

          ${
            dados.pedidoId
              ? `
                <div class="linha">
                  <span>Venta</span>
                  <span>#${escapeHtml(dados.pedidoId)}</span>
                </div>
              `
              : ''
          }

          <div class="linha">
            <span>Cliente</span>
            <span>${escapeHtml(dados.clienteNome)}</span>
          </div>

          <div class="separador"></div>

          ${linhasItens}

          <div class="separador"></div>

          <div class="linha">
            <span>Subtotal</span>
            <span>${formatarMoeda(dados.subtotal)}</span>
          </div>

          <div class="linha">
            <span>Descuento</span>
            <span>${formatarMoeda(dados.desconto)}</span>
          </div>

          <div class="linha">
            <span>Recargo</span>
            <span>${formatarMoeda(dados.acrescimo)}</span>
          </div>

          <div class="linha">
  <span>Base imponible</span>
  <span>${formatarMoeda(dados.baseImponible)}</span>
</div>

<div class="linha">
  <span>Exento / Exonerado</span>
  <span>${formatarMoeda(dados.subtotalExento)}</span>
</div>

<div class="linha">
  <span>IVA 16%</span>
  <span>${formatarMoeda(dados.iva)}</span>
</div>

          <div class="linha total">
            <span>Total</span>
            <span>${formatarMoeda(dados.total)}</span>
          </div>

          <div class="separador"></div>

          ${linhasPagamentos}

          <div class="linha">
            <span>Total pago</span>
            <span>${formatarMoeda(dados.totalPago)}</span>
          </div>

          <div class="linha">
            <span>Cambio</span>
            <span>${formatarMoeda(dados.troco)}</span>
          </div>

          <div class="separador"></div>

          <div class="centro">
            <div>Obrigado por la preferencia</div>
            <div style="margin-top: 10px; 
            font-size: 9px; 
            text-align: center;">
            Impressão: VendaFlow Gestión Comercial</div>
          </div>
        </div>
      </body>
    </html>
  `;
}

function imprimirComprovanteVenda(dados: DadosComprovanteVenda) {
  try {
    const iframe = document.createElement('iframe');

    iframe.style.position = 'fixed';
    iframe.style.right = '0';
    iframe.style.bottom = '0';
    iframe.style.width = '0';
    iframe.style.height = '0';
    iframe.style.border = '0';

    iframe.onload = () => {
      setTimeout(() => {
        iframe.contentWindow?.focus();
        iframe.contentWindow?.print();

        setTimeout(() => {
          iframe.remove();
        }, 1000);
      }, 250);
    };

    iframe.srcdoc = gerarHtmlComprovanteVenda(dados);
    document.body.appendChild(iframe);
  } catch {
    Notify.create({
      type: 'warning',
      message: 'Venta concluída, pero no fue posible abrir la impresión del comprobante',
    });
  }
}

async function carregarClientes() {
  const { data } = await api.get<Cliente[]>('/clientes', {
    params: {
      status: 'ACTIVO',
    },
  });

  clientes.value = data;

  clientesOptions.value = data.map((cliente) => ({
    label: cliente.nome,
    value: cliente.id,
  }));
}

async function carregarProdutos() {
  const { data } = await api.get<Produto[]>('/produtos');
  produtos.value = data;
}

function filtrarClientes(val: string, update: (callback: () => void) => void) {
  update(() => {
    const termo = val.toLowerCase();

    clientesOptions.value = clientes.value
      .filter((cliente) => cliente.nome.toLowerCase().includes(termo))
      .map((cliente) => ({
        label: cliente.nome,
        value: cliente.id,
      }));
  });
}

async function buscarOuAdicionarProduto() {
  const termo = busca.value.trim();

  if (!termo) return;

  const ehNumero = /^\d+$/.test(termo);

  if (!ehNumero) return;

  if (termo.length >= 6) {
    await buscarPorCodigo(termo);
    return;
  }

  await buscarPorIdPro(Number(termo));
}

async function buscarPorIdPro(id: number) {
  try {
    const { data } = await axios.get(`${API_URL}/produtos/${id}`);

    adicionarProduto(data);

    tocarBeep();

    busca.value = '';
    focarBusca();
  } catch {
    beepErro.play();
    Notify.create({
      type: 'negative',
      message: 'Producto no encontrado',
    });

    busca.value = '';
    focarBusca();
  }
}

function focarBusca() {
  inputBusca.value?.focus();
}

async function buscarPorCodigo(codigo: string) {
  try {
    const { data } = await api.get(`/produtos/codigo/${codigo}`);

    adicionarProduto(data);

    tocarBeep();

    busca.value = '';
    focarBusca();
  } catch {
    beepErro.play();
    Notify.create({
      type: 'negative',
      message: 'Producto no encontrado',
    });

    busca.value = '';
    focarBusca();
  }
}

function adicionarProduto(produto: Produto) {
  if (produto.estoque <= 0) {
    beepErro.play();
    Notify.create({
      type: 'warning',
      message: `Producto sin estoque: ${produto.nome}`,
    });
    return;
  }

  const itemExistente = carrinho.value.find((item) => item.produto_id === produto.id);

  if (itemExistente) {
    if (itemExistente.quantidade >= itemExistente.estoqueDisponivel) {
      Notify.create({
        type: 'negative',
        message: `Se ha alcanzado el stock máximo para ${produto.nome}`,
      });
      return;
    }

    tocarBeep();
    itemExistente.quantidade += 1;
    itemExistente.subtotal = itemExistente.quantidade * itemExistente.preco;
  } else {
    carrinho.value.push({
      produto_id: produto.id,
      nome: produto.nome,
      preco: Number(produto.preco),
      quantidade: 1,
      subtotal: Number(produto.preco),
      foto: produto.foto,
      estoqueDisponivel: Number(produto.estoque),
      tipo_imposto: produto.tipo_imposto || 'GRAVADO',
      aliquota_iva: Number(produto.aliquota_iva ?? 16),
    });

    tocarBeep();
    rolarCarrinhoParaVisualizarUltimoItem();
  }
}

function aumentarQuantidade(produtoId: number) {
  const item = carrinho.value.find((i) => i.produto_id === produtoId);
  if (!item) return;

  if (item.quantidade >= item.estoqueDisponivel) {
    Notify.create({
      type: 'warning',
      message: `Se ha alcanzado el stock máximo para ${item.nome}`,
    });
    return;
  }

  item.quantidade += 1;
  item.subtotal = item.quantidade * item.preco;
}

function diminuirQuantidade(produtoId: number) {
  const item = carrinho.value.find((i) => i.produto_id === produtoId);
  if (!item) return;

  if (item.quantidade <= 1) {
    removerItem(produtoId);
    return;
  }

  item.quantidade -= 1;
  item.subtotal = item.quantidade * item.preco;
}

function removerItem(produtoId: number) {
  carrinho.value = carrinho.value.filter((item) => item.produto_id !== produtoId);
}

function calcularFaltanteParaForma(index: number) {
  const pagamentoAtual = pagamentos.value[index];
  if (!pagamentoAtual) return 0;

  const totalOtrosPagos = pagamentos.value.reduce((acc, item, itemIndex) => {
    if (itemIndex === index) return acc;
    return acc + round2(Number(item.valor || 0));
  }, 0);

  return round2(Math.max(0, totalVenda.value - totalOtrosPagos));
}

function onPagamentoChange(indexEditado: number) {
  const pagamento = pagamentos.value[indexEditado];
  if (!pagamento) return;

  const valorInformado = round2(Number(pagamento.valor || 0));

  if (valorInformado < 0) {
    pagamento.valor = null;
    return;
  }

  if (pagamento.forma === 'EFECTIVO') {
    return;
  }

  if (isFormaSemTroco(pagamento.forma)) {
    validarLimiteFormaSemTroco(indexEditado);
  }
}

function validarLimiteFormaSemTroco(indexEditado: number) {
  const pagamento = pagamentos.value[indexEditado];
  if (!pagamento || !isFormaSemTroco(pagamento.forma)) return;

  const totalEfectivoInformado = pagamentos.value
    .filter((item) => item.forma === 'EFECTIVO')
    .reduce((acc, item) => acc + Number(item.valor || 0), 0);

  const totalOutrasFormasSemTroco = pagamentos.value.reduce((acc, item, index) => {
    if (index === indexEditado) return acc;
    if (!isFormaSemTroco(item.forma)) return acc;

    return acc + Number(item.valor || 0);
  }, 0);

  const limite = round2(
    Math.max(0, totalVenda.value - totalEfectivoInformado - totalOutrasFormasSemTroco),
  );

  const valorInformado = round2(Number(pagamento.valor || 0));

  if (valorInformado > limite) {
    pagamento.valor = limite > 0 ? limite : null;

    Notify.create({
      type: 'warning',
      message: `El monto de ${pagamento.label} no puede ser mayor al faltante de ${formatarMoeda(limite)}.`,
    });
    validartarjeta.value = false;
  } else {
    validartarjeta.value = true;
  }
}

function validarTotalSemTroco() {
  const totalSemTroco = pagamentos.value
    .filter((item) => isFormaSemTroco(item.forma))
    .reduce((acc, item) => acc + Number(item.valor || 0), 0);

  if (round2(totalSemTroco) > round2(totalVenda.value)) {
    Notify.create({
      type: 'warning',
      message: `Tarjeta y Pago Móvil no pueden superar el total de la factura: ${formatarMoeda(totalVenda.value)}.`,
    });
  }
}

function ajustarPagamentosSemTroco(indexEditado?: number) {
  let restante = round2(totalVenda.value);

  pagamentos.value.forEach((item, index) => {
    const valorAtual = round2(Number(item.valor || 0));

    if (item.forma === 'EFECTIVO') {
      restante = round2(Math.max(0, restante - valorAtual));
      return;
    }

    if (!isFormaSemTroco(item.forma)) {
      return;
    }

    if (valorAtual > restante) {
      item.valor = restante > 0 ? restante : null;

      // Solo muestra aviso cuando el usuario editó directamente Tarjeta/Pago Móvil
      if (index === indexEditado) {
        Notify.create({
          type: 'warning',
          message: `El monto de ${item.label} no puede ser mayor al faltante de ${formatarMoeda(restante)}.`,
        });
      }
    }

    restante = round2(Math.max(0, restante - Number(item.valor || 0)));
  });
}

async function finalizarVenda() {
  if (carrinho.value.length === 0) {
    Notify.create({
      type: 'warning',
      message: 'Adicione por lo menos un producto',
    });
    return;
  }

  if (!clienteFinalId.value) {
    Notify.create({
      type: 'negative',
      message: 'Cliente padrón no encontrado',
    });
    return;
  }

  const totalVendaAtual = round2(Number(totalVenda.value || 0));

  const pagamentosPayload: PagamentoInformado[] = pagamentos.value
    .map((item) => ({
      forma: item.forma,
      valor: round2(Number(item.valor || 0)),
    }))
    .filter((item) => item.valor > 0);

  if (pagamentosPayload.length === 0) {
    Notify.create({
      type: 'warning',
      message: 'Informe por lo menos una forma de pago',
    });
    return;
  }

  const totalSemTrocoInformado = round2(
    pagamentosPayload
      .filter((item) => isFormaSemTroco(item.forma))
      .reduce((acc, item) => acc + Number(item.valor || 0), 0),
  );

  if (totalSemTrocoInformado > totalVendaAtual) {
    Notify.create({
      type: 'warning',
      message: `Tarjeta y Pago Móvil no pueden superar el total de la factura: ${formatarMoeda(totalVendaAtual)}.`,
    });
    return;
  }

  const totalInformado = round2(
    pagamentosPayload.reduce((acc, item) => acc + Number(item.valor || 0), 0),
  );

  if (totalInformado < totalVendaAtual) {
    Notify.create({
      type: 'warning',
      message: 'El total pago es menor que el total de la venta',
    });
    return;
  }

  const trocoComprovante = round2(Math.max(0, totalInformado - totalVendaAtual));
  const totalPagoComprovante = round2(Math.min(totalInformado, totalVendaAtual));

  const itensComprovante = carrinho.value.map((item) => ({ ...item }));
  const pagamentosComprovante = pagamentosPayload.map((item) => ({ ...item }));

  const clienteNomeComprovante = nomeClienteFinal.value;
  const subtotalComprovante = Number(subtotalVenda.value || 0);
  const descontoComprovante = Number(descontoCalculado.value || 0);
  const acrescimoComprovante = Number(acrescimoCalculado.value || 0);

  const baseImponibleComprovante = Number(baseImponible.value || 0);
  const subtotalExentoComprovante = Number(subtotalExentoAjustado.value || 0);
  const ivaComprovante = Number(ivaCalculado.value || 0);

  const totalComprovante = totalVendaAtual;
  const dataVendaComprovante = new Date();

  const pagamentosFinanceiro: PagamentoInformado[] = pagamentosPayload
    .map((item) => {
      if (item.forma !== 'EFECTIVO') {
        return item;
      }

      return {
        ...item,
        valor: round2(Math.max(0, item.valor - trocoComprovante)),
      };
    })
    .filter((item) => item.valor > 0);

  finalizando.value = true;

  try {
    const formaPagamentoResumo = obterResumoFormaPagamento(pagamentosPayload);

    const { data: pedidoCriado } = await api.post('/pedidos', {
      cliente_id: clienteFinalId.value,
      origem: 'TPV',
      status: 'FINALIZADO',

      desconto: descontoComprovante,
      acrescimo: acrescimoComprovante,

      desconto_tipo: descontoTipo.value,
      desconto_valor: Number(descontoValor.value || 0),

      acrescimo_tipo: acrescimoTipo.value,
      acrescimo_valor: Number(acrescimoValor.value || 0),

      forma_pagamento: formaPagamentoResumo,
      valor_recebido: totalPagoComprovante,
      troco: trocoComprovante,
      pagamentos: pagamentosFinanceiro,

      itens: carrinho.value.map((item) => ({
        produto_id: item.produto_id,
        quantidade: item.quantidade,
      })),
    });

    if (imprimirComprovanteAutomaticamente.value) {
      imprimirComprovanteVenda({
        pedidoId: pedidoCriado?.id || pedidoCriado?.pedido?.id,
        dataVenda: dataVendaComprovante,
        clienteNome: clienteNomeComprovante,
        itens: itensComprovante,
        pagamentos: pagamentosComprovante,
        subtotal: subtotalComprovante,
        desconto: descontoComprovante,
        acrescimo: acrescimoComprovante,
        baseImponible: baseImponibleComprovante,
        subtotalExento: subtotalExentoComprovante,
        iva: ivaComprovante,
        total: totalComprovante,
        totalPago: totalPagoComprovante,
        troco: trocoComprovante,
      });
    }

    Notify.create({
      type: 'positive',
      message: 'Venda realizada com sucesso',
    });

    limparPagamentos();

    carrinho.value = [];
    clienteSelecionado.value = null;
    busca.value = '';

    descontoTipo.value = 'valor';
    descontoValor.value = 0;

    acrescimoTipo.value = 'valor';
    acrescimoValor.value = 0;

    modalFaturar.value = false;

    await carregarProdutos();
    focarBusca();
  } catch (error: unknown) {
    let mensagem = 'Error al finalizar venta';

    if (axios.isAxiosError(error)) {
      mensagem = error.response?.data?.erro || mensagem;
    }

    Notify.create({
      type: 'negative',
      message: mensagem,
    });
  } finally {
    finalizando.value = false;
  }
}

watch(totalVenda, () => {
  ajustarPagamentosSemTroco();
});

function limparPagamentos() {
  pagamentos.value = [
    { forma: 'EFECTIVO', label: 'Efectivo', valor: null },
    { forma: 'TARJETA', label: 'Tarjeta', valor: null },
    { forma: 'PAGOMOVIL', label: 'Pago Movil', valor: null },
  ];
}

const scrollCarrinhoRef = ref<HTMLElement | null>(null);

function rolarCarrinhoParaVisualizarUltimoItem() {
  nextTick(() => {
    const container = scrollCarrinhoRef.value;
    if (!container) return;

    const ultimoItem = container.lastElementChild as HTMLElement | null;
    if (!ultimoItem) return;

    ultimoItem.scrollIntoView({
      behavior: 'smooth',
      block: 'nearest',
    });
  });
}

watch([descontoTipo, descontoValor], ([tipo, valor]) => {
  if (tipo === 'percentual' && Number(valor) > 100) {
    descontoValor.value = 100;
  }
});

watch([acrescimoTipo, acrescimoValor], ([tipo, valor]) => {
  if (tipo === 'percentual' && Number(valor) > 100) {
    acrescimoValor.value = 100;
  }
});

onMounted(async () => {
  await Promise.all([carregarClientes(), carregarProdutos()]);
  focarBusca();
});
</script>

<style scoped>
.sticky-card {
  position: sticky;
  top: 20px;
}

@media (max-width: 1023px) {
  .sticky-card {
    position: static;
  }
}

.border {
  border-radius: 12px;
}

.lista-scroll {
  max-height: calc(5 * 60px);
  overflow-y: auto;
}

.lista-produtos {
  height: 280px;
  overflow-y: scroll;
  border: 1px solid #ddd;
}

.scroll-carrinho {
  max-height: 200px;
  overflow-y: auto;
}

:deep(.sem-setas input::-webkit-outer-spin-button),
:deep(.sem-setas input::-webkit-inner-spin-button) {
  -webkit-appearance: none;
  margin: 0;
}

:deep(.sem-setas input[type='number']) {
  -moz-appearance: textfield;
  appearance: textfield;
}

.subir {
  margin-top: -7px;
}
</style>
