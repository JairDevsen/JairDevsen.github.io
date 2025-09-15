<?php
// filepath: c:\projetos\dentalcare_duo\PHP\agenda.php
require 'conexao.php';
require 'session_check.php';

// Gera os próximos 7 dias (incluindo hoje, exceto domingos)
$hoje = date('Y-m-d');
$hoje_w = date('w', strtotime($hoje));
// Se hoje for domingo, começa na próxima segunda
if ($hoje_w == 0) {
    $data = date('Y-m-d', strtotime($hoje . ' +1 day'));
} else {
    $data = $hoje;
}
$dias = [];
while (count($dias) < 7) {
    $w = date('w', strtotime($data));
    if ($w != 0) { // inclui segunda a sábado, nunca domingo
        $dias[] = $data;
    }
    $data = date('Y-m-d', strtotime($data . ' +1 day'));
}

// Busca horários da semana
$stmt = $pdo->prepare("SELECT * FROM horarios WHERE data IN (" . implode(",", array_fill(0, count($dias), '?')) . ") ORDER BY data, hora");
$stmt->execute($dias);
$horarios = $stmt->fetchAll(PDO::FETCH_ASSOC);

// Organiza por dia/hora
$agenda = [];
foreach ($dias as $dia) {
    for ($h = 8; $h <= 17; $h++) {
        $hora = sprintf('%02d:00:00', $h);
        $agenda[$dia][$hora] = 'livre';
    }
}
foreach ($horarios as $h) {
    $agenda[$h['data']][$h['hora']] = $h['status'];
}

// Atualização de status (apenas admin)
if (is_admin() && isset($_POST['status'], $_POST['data'], $_POST['hora'])) {
    $data = $_POST['data'];
    $hora = $_POST['hora'];
    $novo = $_POST['status'] === 'livre' ? 'ocupado' : 'livre';
    // Verifica se já existe
    $stmt = $pdo->prepare("SELECT id FROM horarios WHERE data = ? AND hora = ?");
    $stmt->execute([$data, $hora]);
    $row = $stmt->fetch(PDO::FETCH_ASSOC);
    if ($row) {
        $id = $row['id'];
        $stmt = $pdo->prepare("UPDATE horarios SET status = ? WHERE id = ?");
        $stmt->execute([$novo, $id]);
    } else {
        $stmt = $pdo->prepare("INSERT INTO horarios (data, hora, status) VALUES (?, ?, ?)");
        $stmt->execute([$data, $hora, $novo]);
    }
    header('Location: agenda.php');
    exit;
}

// Função para nome do dia (corrige acentuação do Windows)
function nomeDia($data) {
    $formatter = new IntlDateFormatter(
        'pt_BR',
        IntlDateFormatter::FULL,
        IntlDateFormatter::NONE,
        'America/Sao_Paulo',
        IntlDateFormatter::GREGORIAN,
        'EEEE'
    );
    return ucfirst($formatter->format(new DateTime($data)));
}


// Função para formatar data
function formatarData($data) {
    return date('d/m/Y', strtotime($data));
}
setlocale(LC_TIME, 'pt_BR', 'pt_BR.utf-8', 'portuguese');
?>
<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <title>Agenda Semanal - DentalCare Duo</title>
    <link rel="stylesheet" href="/dentalcare_duo/css/main.css">
    <link rel="stylesheet" href="/dentalcare_duo/css/agenda.css">
</head>
<body>
    <!-- Header igual ao das outras páginas -->
    <!-- ...copie o header de services_overview.html... -->
     <header class="bg-white shadow-soft sticky top-0 z-50">
        <nav class="container mx-auto px-4 py-4">
            <div class="flex items-center justify-between">
                <!-- Logo -->
                <div class="flex items-center space-x-3">
                    <div class="w-12 h-12 bg-primary-600 rounded-xl flex items-center justify-center">
                        <svg class="w-8 h-8 text-white" fill="currentColor" viewBox="0 0 24 24">
                            <path d="M12 2C6.48 2 2 6.48 2 12s4.48 10 10 10 10-4.48 10-10S17.52 2 12 2zm-2 15l-5-5 1.41-1.41L10 14.17l7.59-7.59L19 8l-9 9z"/>
                        </svg>
                    </div>
                    <div>
                        <h1 class="text-xl font-heading font-bold text-primary-600">DentalCare Duo</h1>
                        <p class="text-xs text-text-secondary">Saloá & Garanhuns</p>
                    </div>
                </div>

                <!-- Desktop Navigation -->

                <div class="hidden md:flex items-center space-x-8">
                     <a href="../pages/homepage.html" class="text-text-primary hover:text-primary-600 transition-fast">Início</a>
                      <a href="../pages/about_dr_name.html" class="text-text-primary hover:text-primary-600 transition-fast">Sobre o Doutor</a>
                      <a href="../pages/services_overview.html" class="text-text-primary hover:text-primary-600 transition-fast">Serviços</a>
                      <a href="agenda.php" class="text-primary-600 font-semibold border-b-2 border-primary-600 pb-1">Agenda</a>
                      <a href="login.php" class="text-text-primary hover:text-primary-600 transition-fast">Login</a>
                      <a href="javascript:void(0)" class="text-text-primary hover:text-primary-600 transition-fast">Contato</a>
                </div>


                <!-- Location Toggle & CTA -->
                <div class="hidden md:flex items-center space-x-4">
                   
                    <a href="javascript:void(0)" class="btn btn-primary">Agendar Consulta</a>
                </div>

                              <!-- Mobile Menu Button -->
                <button id="mobile-menu-btn" class="md:hidden p-2">
                    <svg class="w-6 h-6" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 6h16M4 12h16M4 18h16"/>
                    </svg>
                </button>
            </div>
            <!-- Mobile Menu -->
            <div id="mobile-menu" class="md:hidden hidden mt-4 pb-4 border-t border-surface-200">
                <div class="flex flex-col space-y-4 mt-4">
                    <a href="../pages/homepage.html" class="text-text-primary">Início</a>
                    <a href="../pages/about_dr_name.html" class="text-text-primary">Sobre o Doutor</a>
                    <a href="../pages/services_overview.html" class="text-text-primary">Serviços</a>
                    <a href="agenda.php" class="text-text-primary">Agenda</a>
                    <a href="login.php" class="text-text-primary">Login</a>
                    <a href="javascript:void(0)" class="text-text-primary">Contato</a>
                    <a href="javascript:void(0)" class="btn btn-primary w-full">Agendar Consulta</a>
                </div>
            </div>
        </nav>
    </header>

    <!-- Page Header -->

    <!-- Page Header -->
    <section class="hero-gradient text-white py-12 md:py-16">
        <div class="container mx-auto px-4">
            <div class="text-center max-w-4xl mx-auto">
                <h1 class="text-3xl md:text-5xl font-heading font-bold mb-6">
                    Agenda
                </h1>
            </div>
        </div>
    </section>
    <div class="container mx-auto px-4">
        <?php if (is_admin()): ?>
            <a href="logout.php" class="btn bg-error-600 text-white mb-4 float-right">Logout</a>
        <?php endif; ?>
        <table>
            <thead>
                <tr>
                    <th>Hora</th>
                    <?php foreach ($dias as $dia): ?>
                        <th><?= nomeDia($dia); ?><br><?= formatarData($dia) ?></th>
                    <?php endforeach; ?>
                </tr>
            </thead>
            <tbody>
            <?php for ($h = 8; $h <= 17; $h++): 
                $hora = sprintf('%02d:00:00', $h);
                $horaLabel = sprintf('%02d:00', $h);
            ?>
<tr>
    <td><strong><?= $horaLabel ?></strong></td>
    <?php foreach ($dias as $dia): 
        $status = $agenda[$dia][$hora];
        $msg = "Olá, gostaria de marcar uma consulta no dia ".formatarData($dia)." às ".$horaLabel.".";
        $wa = "https://wa.me/5587999999999?text=".urlencode($msg);
        // Busca o id do horário
        $stmt = $pdo->prepare("SELECT id FROM horarios WHERE data = ? AND hora = ?");
        $stmt->execute([$dia, $hora]);
        $row = $stmt->fetch(PDO::FETCH_ASSOC);
        $id = $row ? $row['id'] : null;
    ?>
    <td>
        <?php if (is_admin()): ?>
            <form method="post" style="display:inline;">
                <input type="hidden" name="id" value="<?= $id ?? 0 ?>">
                <input type="hidden" name="status" value="<?= $status ?>">
                <input type="hidden" name="data" value="<?= $dia ?>">
                <input type="hidden" name="hora" value="<?= $hora ?>">
                <button type="submit" class="btn <?= $status=='livre' ? 'bg-success-600' : 'bg-error-600' ?> text-white w-full">
                    <?= ucfirst($status) ?>
                </button>
            </form>
        <?php else: ?>
            <?php if ($status == 'livre'): ?>
                <a href="<?= $wa ?>" target="_blank" class="btn-livre">Livre</a>
            <?php else: ?>
                <button class="btn-ocupado" disabled>Ocupado</button>
            <?php endif; ?>
        <?php endif; ?>
    </td>
    <?php endforeach; ?>
</tr>
<?php endfor; ?>
            </tbody>
        </table>
    </div>
    <script>
        // Mobile menu toggle
        const mobileMenuBtn = document.getElementById('mobile-menu-btn');
        const mobileMenu = document.getElementById('mobile-menu');
        if (mobileMenuBtn && mobileMenu) {
            mobileMenuBtn.addEventListener('click', () => {
                mobileMenu.classList.toggle('hidden');
            });
        }
    </script>
</body>
</html>
