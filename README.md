from enum import Enum
from dataclasses import dataclass
from typing import List, Optional, Dict, Tuple
from itertools import combinations
from collections import Counter
import sys

class DiceRule(Enum):
    ONE = 0
    TWO = 1
    THREE = 2
    FOUR = 3
    FIVE = 4
    SIX = 5
    CHOICE = 6
    FOUR_OF_A_KIND = 7
    FULL_HOUSE = 8
    SMALL_STRAIGHT = 9
    LARGE_STRAIGHT = 10
    YACHT = 11

@dataclass
class Bid:
    group: str
    amount: int

@dataclass
class DicePut:
    rule: DiceRule
    dice: List[int]

class Game:
    def __init__(self):
        self.my_state = GameState()
        self.opp_state = GameState()
        self.round = 0
        self.opp_bid_amounts = []
        self.opp_bid_groups = []
        self.my_bid_history = []
        self.dice_history = []
        self.opp_rule_priorities = []
        self.bid_conflicts = []
        self.opp_puts_history = []
        self.opp_bid_success_rate = 0.0
        self.my_bid_success_rate = 0.0
        self.aggressive_threshold = 800  
        self.is_aggressive_opponent = False
        self.opponent_pattern = "unknown"
        self.last_3_opp_bids = []
        self.bidding_wars_won = 0
        self.bidding_wars_total = 0
        self.opp_high_value_targeting = False
        self.my_recent_losses = 0
        self.confirmed_ai_type = None
        
    def detect_opponent_ai_type_precise(self) -> str:
        """정확한 상대 AI 타입 탐지 - 로그 기반으로 개선"""
        if len(self.opp_bid_amounts) < 2:
            return "unknown"
            
        zero_bids = sum(1 for bid in self.opp_bid_amounts if bid == 0)
        zero_rate = zero_bids / len(self.opp_bid_amounts) if self.opp_bid_amounts else 0
        avg_bid = sum(self.opp_bid_amounts) / len(self.opp_bid_amounts) if self.opp_bid_amounts else 0
        max_bid = max(self.opp_bid_amounts) if self.opp_bid_amounts else 0
        
        # 로그 분석 결과에 따른 정교한 AI 분류
        if zero_rate >= 0.95 and max_bid <= 100:
            return "sample_ai_3_4_5_8"  # 거의 항상 0점 입찰
        elif zero_rate >= 0.8 and max_bid <= 500:
            if any(rule >= 6 for rule in self.opp_rule_priorities[-3:] if self.opp_rule_priorities):
                return "sample_ai_8"  # 조합 우선
            elif any(rule < 6 for rule in self.opp_rule_priorities[-3:] if self.opp_rule_priorities):
                return "sample_ai_5"  # 기본 우선
            else:
                return "sample_ai_3_4"  # 기타 수동적
        elif avg_bid >= 3000 and any(bid >= 10000 for bid in self.opp_bid_amounts):
            return "sample_ai_7_10"  # 매우 공격적
        elif avg_bid >= 1500 and max_bid >= 5000:
            return "sample_ai_6_9"  # 중간 공격적
        elif avg_bid >= 1000 and "difference_pattern" in str(self.opp_bid_amounts):
            return "sample_ai_2"  # 차이 기반
        elif zero_rate <= 0.7 and avg_bid >= 200:
            return "sample_ai_1"  # 기본 예제
        else:
            # 더 정교한 분석
            recent_bids = self.opp_bid_amounts[-5:] if len(self.opp_bid_amounts) >= 5 else self.opp_bid_amounts
            high_bids = [b for b in recent_bids if b >= 2000]
            
            if len(high_bids) >= 3:
                return "sample_ai_7_10"
            elif len(high_bids) >= 2:
                return "sample_ai_6_9"
            elif zero_rate >= 0.7:
                return "sample_ai_3_4_5_8"
            else:
                return "sample_ai_1_2"

    def calculate_bid_targeted(self, dice_a: List[int], dice_b: List[int]) -> Bid:
        """각 AI별 맞춤형 입찰 전략"""
        # AI 타입 확정
        if self.confirmed_ai_type is None:
            self.confirmed_ai_type = self.detect_opponent_ai_type_precise()
        
        # 주사위 평가
        eval_a = self.evaluate_dice_comprehensive(dice_a)
        eval_b = self.evaluate_dice_comprehensive(dice_b)
        
        # 게임 상황
        rounds_left = 13 - self.round
        my_score = self.my_state.get_total_score()
        opp_score = self.opp_state.get_total_score()
        score_diff = my_score - opp_score
        
        # 그룹 선택
        if eval_a["total_value"] > eval_b["total_value"]:
            preferred_group = "A"
            value_diff = eval_a["total_value"] - eval_b["total_value"]
            my_eval = eval_a
            alt_eval = eval_b
        else:
            preferred_group = "B" 
            value_diff = eval_b["total_value"] - eval_a["total_value"]
            my_eval = eval_b
            alt_eval = eval_a
        
        # AI별 맞춤 전략
        if self.confirmed_ai_type == "sample_ai_1" or "sample_ai_1" in self.confirmed_ai_type:
            bid_amount = self._bid_vs_ai1(my_eval, value_diff, score_diff, rounds_left)
        elif self.confirmed_ai_type == "sample_ai_2" or "sample_ai_2" in self.confirmed_ai_type:
            bid_amount = self._bid_vs_ai2(dice_a, dice_b, my_eval, value_diff, score_diff)
        elif "sample_ai_3" in self.confirmed_ai_type or "sample_ai_4" in self.confirmed_ai_type:
            bid_amount = self._bid_vs_ai3_4(my_eval, value_diff, rounds_left)
        elif self.confirmed_ai_type == "sample_ai_5":
            bid_amount = self._bid_vs_ai5(my_eval, value_diff, rounds_left)
        elif self.confirmed_ai_type == "sample_ai_6" or "sample_ai_6" in self.confirmed_ai_type:
            bid_amount = self._bid_vs_ai6(my_eval, value_diff, score_diff, rounds_left)
        elif self.confirmed_ai_type == "sample_ai_7" or "sample_ai_7" in self.confirmed_ai_type:
            bid_amount = self._bid_vs_ai7(my_eval, value_diff, score_diff, rounds_left)
        elif self.confirmed_ai_type == "sample_ai_8":
            bid_amount = self._bid_vs_ai8(my_eval, value_diff, rounds_left)
        elif self.confirmed_ai_type == "sample_ai_9" or "sample_ai_9" in self.confirmed_ai_type:
            bid_amount = self._bid_vs_ai9(my_eval, value_diff, score_diff, rounds_left)
        elif self.confirmed_ai_type == "sample_ai_10" or "sample_ai_10" in self.confirmed_ai_type:
            bid_amount = self._bid_vs_ai10(my_eval, value_diff, score_diff, rounds_left)
        else:
            # 일반 전략
            bid_amount = self._bid_general_safe(my_eval, value_diff, score_diff, rounds_left)
        
        return Bid(preferred_group, min(100000, max(0, bid_amount)))

    def _bid_vs_ai1(self, my_eval: Dict, value_diff: float, score_diff: int, rounds_left: int) -> int:
        """AI #1 대응: 기본 예제 AI - 점수차/10 기반 입찰"""
        if my_eval["combo_value"] >= 40000:  # 매우 좋은 조합
            return min(15000, int(value_diff * 0.4))
        elif my_eval["combo_value"] >= 20000:
            return min(8000, int(value_diff * 0.3))
        elif my_eval["basic_value"] >= 15000:
            return min(5000, int(value_diff * 0.25))
        elif value_diff >= 15000:
            return min(3000, int(value_diff * 0.2))
        else:
            return min(1000, int(value_diff * 0.1))

    def _bid_vs_ai2(self, dice_a: List[int], dice_b: List[int], my_eval: Dict, value_diff: float, score_diff: int) -> int:
        """AI #2 대응: 주사위 합 차이 * 1000 입찰"""
        sum_a = sum(dice_a)
        sum_b = sum(dice_b)
        dice_sum_diff = abs(sum_a - sum_b)
        
        # AI #2는 주사위 합 차이에 1000을 곱해서 입찰
        expected_opp_bid = dice_sum_diff * 1000
        
        if my_eval["combo_value"] >= 40000:
            return min(expected_opp_bid + 8000, int(value_diff * 0.6))
        elif my_eval["combo_value"] >= 25000:
            return min(expected_opp_bid + 5000, int(value_diff * 0.5))
        elif value_diff >= 20000:
            return min(expected_opp_bid + 3000, int(value_diff * 0.4))
        else:
            return min(expected_opp_bid + 1000, int(value_diff * 0.2))

    def _bid_vs_ai3_4(self, my_eval: Dict, value_diff: float, rounds_left: int) -> int:
        """AI #3,#4 대응: 항상 0점 입찰하는 수동적 AI"""
        # 0점만 입찰하므로 최소 입찰로도 승리
        if value_diff >= 30000:
            return min(500, int(value_diff * 0.02))
        elif value_diff >= 15000:
            return min(200, int(value_diff * 0.015))
        elif value_diff >= 5000:
            return min(100, int(value_diff * 0.01))
        else:
            return 0

    def _bid_vs_ai5(self, my_eval: Dict, value_diff: float, rounds_left: int) -> int:
        """AI #5 대응: 기본 점수 규칙 우선 + 0점 입찰"""
        if value_diff >= 30000:
            return min(800, int(value_diff * 0.03))
        elif value_diff >= 15000:
            return min(400, int(value_diff * 0.025))
        elif value_diff >= 5000:
            return min(200, int(value_diff * 0.02))
        else:
            return 0

    def _bid_vs_ai6(self, my_eval: Dict, value_diff: float, score_diff: int, rounds_left: int) -> int:
        """AI #6 대응: 수비적 + 기본 규칙 우선"""
        # 로그에서 보면 중간 정도의 입찰을 함
        if my_eval["combo_value"] >= 50000:
            return min(25000, int(value_diff * 0.7))
        elif my_eval["combo_value"] >= 30000:
            return min(18000, int(value_diff * 0.6))
        elif my_eval["combo_value"] >= 20000:
            return min(12000, int(value_diff * 0.5))
        elif my_eval["basic_value"] >= 18000:
            return min(8000, int(value_diff * 0.4))
        elif value_diff >= 15000:
            return min(5000, int(value_diff * 0.3))
        else:
            return min(2000, int(value_diff * 0.15))

    def _bid_vs_ai7(self, my_eval: Dict, value_diff: float, score_diff: int, rounds_left: int) -> int:
        """AI #7 대응: 공격적 + 기본 규칙 우선"""
        # 로그에서 보면 꽤 높은 입찰을 함
        if my_eval["combo_value"] >= 50000:
            return min(45000, int(value_diff * 0.9))
        elif my_eval["combo_value"] >= 30000:
            return min(35000, int(value_diff * 0.8))
        elif my_eval["combo_value"] >= 20000:
            return min(25000, int(value_diff * 0.7))
        elif my_eval["basic_value"] >= 18000:
            return min(18000, int(value_diff * 0.6))
        elif value_diff >= 15000:
            return min(12000, int(value_diff * 0.5))
        else:
            return min(6000, int(value_diff * 0.3))

    def _bid_vs_ai8(self, my_eval: Dict, value_diff: float, rounds_left: int) -> int:
        """AI #8 대응: 조합 규칙 우선 + 0점 입찰"""
        if value_diff >= 30000:
            return min(1000, int(value_diff * 0.04))
        elif value_diff >= 15000:
            return min(500, int(value_diff * 0.03))
        elif value_diff >= 5000:
            return min(300, int(value_diff * 0.02))
        else:
            return 0

    def _bid_vs_ai9(self, my_eval: Dict, value_diff: float, score_diff: int, rounds_left: int) -> int:
        """AI #9 대응: 수비적 + 조합 규칙 우선"""
        if my_eval["combo_value"] >= 50000:
            return min(35000, int(value_diff * 0.8))
        elif my_eval["combo_value"] >= 30000:
            return min(25000, int(value_diff * 0.7))
        elif my_eval["combo_value"] >= 20000:
            return min(18000, int(value_diff * 0.6))
        elif value_diff >= 20000:
            return min(12000, int(value_diff * 0.5))
        elif value_diff >= 10000:
            return min(6000, int(value_diff * 0.4))
        else:
            return min(3000, int(value_diff * 0.2))

    def _bid_vs_ai10(self, my_eval: Dict, value_diff: float, score_diff: int, rounds_left: int) -> int:
        """AI #10 대응: 공격적 + 조합 규칙 우선"""
        if my_eval["combo_value"] >= 50000:
            return min(50000, int(value_diff * 1.0))
        elif my_eval["combo_value"] >= 30000:
            return min(40000, int(value_diff * 0.9))
        elif my_eval["combo_value"] >= 20000:
            return min(30000, int(value_diff * 0.8))
        elif value_diff >= 25000:
            return min(20000, int(value_diff * 0.7))
        elif value_diff >= 15000:
            return min(15000, int(value_diff * 0.6))
        else:
            return min(8000, int(value_diff * 0.4))

    def _bid_general_safe(self, my_eval: Dict, value_diff: float, score_diff: int, rounds_left: int) -> int:
        """일반적인 안전한 입찰 전략"""
        if my_eval["combo_value"] >= 40000:
            return min(20000, int(value_diff * 0.5))
        elif my_eval["combo_value"] >= 25000:
            return min(15000, int(value_diff * 0.4))
        elif value_diff >= 20000:
            return min(10000, int(value_diff * 0.3))
        else:
            return min(5000, int(value_diff * 0.2))

    def evaluate_dice_comprehensive(self, dice: List[int]) -> Dict:
        """포괄적인 주사위 평가"""
        if not dice:
            return {"total_value": 0, "basic_value": 0, "combo_value": 0}
            
        counter = Counter(dice)
        unique_dice = sorted(set(dice))
        max_count = max(counter.values()) if counter else 0
        
        # 기본 점수들
        basic_scores = {}
        for i in range(6):
            basic_scores[i] = counter.get(i + 1, 0) * (i + 1) * 1000
            
        # 조합 점수들
        combo_scores = {}
        combo_scores[6] = sum(dice) * 1000  # CHOICE
        combo_scores[7] = sum(dice) * 1000 if max_count >= 4 else 0  # FOUR_OF_A_KIND
        
        # FULL_HOUSE
        counts = sorted(counter.values(), reverse=True)
        if (len(counts) >= 2 and counts[0] >= 3 and counts[1] >= 2) or counts[0] == 5:
            combo_scores[8] = sum(dice) * 1000
        else:
            combo_scores[8] = 0
            
        combo_scores[9] = 15000 if self._has_small_straight(unique_dice) else 0
        combo_scores[10] = 30000 if self._has_large_straight(unique_dice) else 0
        combo_scores[11] = 50000 if max_count == 5 else 0
        
        # 사용 가능한 규칙만 고려
        unused_rules = [i for i, score in enumerate(self.my_state.rule_score) if score is None]
        
        usable_basic_scores = {k: v for k, v in basic_scores.items() if k in unused_rules}
        usable_combo_scores = {k: v for k, v in combo_scores.items() if k in unused_rules}
        
        max_basic = max(usable_basic_scores.values()) if usable_basic_scores else 0
        max_combo = max(usable_combo_scores.values()) if usable_combo_scores else 0
        
        # 보너스 기여도
        my_upper = self.my_state.get_upper_section_score() or 0
        basic_rules_left = len([r for r in unused_rules if r < 6])
        bonus_contribution = 0
        
        if basic_rules_left > 0 and my_upper < 63000:
            remaining_for_bonus = 63000 - my_upper
            if remaining_for_bonus > 0:
                bonus_contribution = min(35000, remaining_for_bonus) / basic_rules_left
        
        total_value = max_basic + max_combo + bonus_contribution + len(unused_rules) * 500
        
        return {
            "total_value": total_value,
            "basic_value": max_basic,
            "combo_value": max_combo,
            "bonus_contribution": bonus_contribution,
            "sum": sum(dice),
            "max_count": max_count,
            "flexibility": len(unused_rules)
        }

    def calculate_put_targeted(self) -> DicePut:
        """각 AI별 맞춤형 주사위 배치 전략"""
        dice_arr = self.my_state.dice
        
        if not dice_arr:
            return DicePut(DiceRule.CHOICE, [1, 1, 1, 1, 1])
        
        unused_rules = [i for i, score in enumerate(self.my_state.rule_score) if score is None]
        if not unused_rules:
            safe_dice = dice_arr[:5] if len(dice_arr) >= 5 else dice_arr + [6] * (5 - len(dice_arr))
            return DicePut(DiceRule.CHOICE, safe_dice)
        
        # AI별 우선순위 결정
        if self.confirmed_ai_type in ["sample_ai_5", "sample_ai_6", "sample_ai_7"]:
            # 기본 규칙 우선 AI들
            priority_rules = self._get_basic_priority_rules(unused_rules, dice_arr)
        elif self.confirmed_ai_type in ["sample_ai_8", "sample_ai_9", "sample_ai_10"]:
            # 조합 규칙 우선 AI들
            priority_rules = self._get_combo_priority_rules(unused_rules, dice_arr)
        else:
            # 기타 AI들 - 균형잡힌 접근
            priority_rules = self._get_balanced_priority_rules(unused_rules, dice_arr)
        
        best_score = -200000
        best_put = None
        
        for rule_idx in priority_rules:
            rule = DiceRule(rule_idx)
            best_combo = self._find_optimal_combination(rule, dice_arr)
            
            if best_combo and self._is_valid_combination(best_combo, dice_arr):
                base_score = GameState.calculate_score(DicePut(rule, best_combo))
                strategic_score = self._calculate_strategic_value(rule, base_score, unused_rules)
                
                if strategic_score > best_score:
                    best_score = strategic_score
                    best_put = DicePut(rule, best_combo)
        
        if best_put is None:
            rule = DiceRule(unused_rules[0])
            safe_dice = self._make_safe_combination(dice_arr, rule)
            best_put = DicePut(rule, safe_dice)
        
        return best_put

    def _get_basic_priority_rules(self, unused_rules: List[int], dice_arr: List[int]) -> List[int]:
        """기본 규칙 우선 정렬"""
        counter = Counter(dice_arr)
        
        # 기본 규칙을 점수 높은 순으로
        basic_rules = [(rule, counter.get(rule + 1, 0) * (rule + 1) * 1000) 
                      for rule in unused_rules if rule < 6]
        basic_rules.sort(key=lambda x: x[1], reverse=True)
        
        # 조합 규칙을 잠재 점수순으로
        combo_rules = []
        for rule in unused_rules:
            if rule >= 6:
                score = self._estimate_combo_score(rule, dice_arr)
                combo_rules.append((rule, score))
        combo_rules.sort(key=lambda x: x[1], reverse=True)
        
        return [rule for rule, score in basic_rules] + [rule for rule, score in combo_rules]

    def _get_combo_priority_rules(self, unused_rules: List[int], dice_arr: List[int]) -> List[int]:
        """조합 규칙 우선 정렬"""
        counter = Counter(dice_arr)
        
        # 조합 규칙을 잠재 점수순으로
        combo_rules = []
        for rule in unused_rules:
            if rule >= 6:
                score = self._estimate_combo_score(rule, dice_arr)
                combo_rules.append((rule, score))
        combo_rules.sort(key=lambda x: x[1], reverse=True)
        
        # 기본 규칙을 점수 높은 순으로
        basic_rules = [(rule, counter.get(rule + 1, 0) * (rule + 1) * 1000) 
                      for rule in unused_rules if rule < 6]
        basic_rules.sort(key=lambda x: x[1], reverse=True)
        
        return [rule for rule, score in combo_rules] + [rule for rule, score in basic_rules]

    def _get_balanced_priority_rules(self, unused_rules: List[int], dice_arr: List[int]) -> List[int]:
        """균형잡힌 규칙 우선순위"""
        all_rules = []
        counter = Counter(dice_arr)
        
        for rule in unused_rules:
            if rule < 6:
                score = counter.get(rule + 1, 0) * (rule + 1) * 1000
            else:
                score = self._estimate_combo_score(rule, dice_arr)
            all_rules.append((rule, score))
        
        all_rules.sort(key=lambda x: x[1], reverse=True)
        return [rule for rule, score in all_rules]

    def _estimate_combo_score(self, rule: int, dice_arr: List[int]) -> int:
        """조합 규칙의 예상 점수"""
        counter = Counter(dice_arr)
        unique_dice = sorted(set(dice_arr))
        max_count = max(counter.values()) if counter else 0
        
        if rule == 6:  # CHOICE
            return sum(dice_arr) * 1000
        elif rule == 7:  # FOUR_OF_A_KIND
            return sum(dice_arr) * 1000 if max_count >= 4 else 0
        elif rule == 8:  # FULL_HOUSE
            counts = sorted(counter.values(), reverse=True)
            if (len(counts) >= 2 and counts[0] >= 3 and counts[1] >= 2) or counts[0] == 5:
                return sum(dice_arr) * 1000
            else:
                return 0
        elif rule == 9:  # SMALL_STRAIGHT
            return 15000 if self._has_small_straight(unique_dice) else 0
        elif rule == 10:  # LARGE_STRAIGHT
            return 30000 if self._has_large_straight(unique_dice) else 0
        elif rule == 11:  # YACHT
            return 50000 if max_count == 5 else 0
        return 0

    def _calculate_strategic_value(self, rule: DiceRule, base_score: int, unused_rules: List[int]) -> float:
        """전략적 가치 계산"""
        if base_score < 0:
            return -100000.0
            
        value = float(base_score)
        
        # 보너스 상황 고려
        bonus_urgency = self._calculate_bonus_urgency()
        if bonus_urgency >= 3 and rule.value < 6:
            value += 100000
        elif bonus_urgency >= 2 and rule.value < 6:
            value += 60000
        elif bonus_urgency >= 1 and rule.value < 6:
            value += 30000
            
        # 희귀 조합 보너스
        if rule == DiceRule.YACHT and base_score > 0:
            value += 150000
        elif rule == DiceRule.LARGE_STRAIGHT and base_score > 0:
            value += 100000
        elif rule == DiceRule.FOUR_OF_A_KIND and base_score > 0:
            value += 80000
        elif rule == DiceRule.FULL_HOUSE and base_score > 0:
            value += 70000
        elif rule == DiceRule.SMALL_STRAIGHT and base_score > 0:
            value += 50000
            
        # 유연성 보너스
        value += len(unused_rules) * 1000
        
        return value

    def _calculate_bonus_urgency(self) -> int:
        """보너스 달성 긴급도 (0-5)"""
        my_upper = self.my_state.get_upper_section_score() or 0
        unused_basic = [i for i in range(6) if self.my_state.rule_score[i] is None]
        
        if not unused_basic:
            return 0
            
        remaining = 63000 - my_upper
        if remaining <= 0:
            return 0
            
        avg_needed = remaining / len(unused_basic)
        rounds_left = 13 - self.round
        
        if avg_needed <= 2000:
            return 5
        elif avg_needed <= 3000:
            return 4
        elif avg_needed <= 5000:
            return 3
        elif avg_needed <= 8000 and rounds_left <= 6:
            return 2
        elif avg_needed <= 12000:
            return 1
        else:
            return 0

    def _find_optimal_combination(self, rule: DiceRule, dice_arr: List[int]) -> List[int]:
        """최적 조합 찾기"""
        if len(dice_arr) <= 5:
            result = dice_arr[:5]
            while len(result) < 5:
                result.append(6)
            return result
        
        counter = Counter(dice_arr)
        
        if rule.value < 6:  # 기본 점수 규칙
            target_num = rule.value + 1
            target_count = counter.get(target_num, 0)
            
            # 타겟 숫자를 최대한 포함
            result = [target_num] * min(5, target_count)
            
            # 남은 자리는 가장 높은 숫자들로
            if len(result) < 5:
                others = []
                for num in sorted(counter.keys(), reverse=True):
                    if num != target_num:
                        others.extend([num] * counter[num])
                
                needed = 5 - len(result)
                result.extend(others[:needed])
                
            while len(result) < 5:
                result.append(6)
                
            return result[:5]
            
        elif rule == DiceRule.YACHT:
            # 가장 많은 수 선택
            if counter:
                most_common = counter.most_common()
                best_num = most_common[0][0]
                best_count = most_common[0][1]
                
                if best_count >= 4:
                    result = [best_num] * 5
                else:
                    # 가장 높은 수로 선택
                    highest_num = max(counter.keys())
                    result = [highest_num] * 5
                return result
            return [6] * 5
            
        elif rule == DiceRule.LARGE_STRAIGHT:
            target_straights = [[1,2,3,4,5], [2,3,4,5,6]]
            for straight in target_straights:
                if all(counter.get(num, 0) > 0 for num in straight):
                    return straight
            return [2,3,4,5,6]
            
        elif rule == DiceRule.SMALL_STRAIGHT:
            target_straights = [[1,2,3,4], [2,3,4,5], [3,4,5,6]]
            for straight in target_straights:
                if all(counter.get(num, 0) > 0 for num in straight):
                    result = straight.copy()
                    remaining = [d for d in dice_arr if d not in straight]
                    if remaining:
                        result.append(max(remaining))
                    else:
                        result.append(6)
                    return result
            return [3,4,5,6,6]
            
        elif rule == DiceRule.FOUR_OF_A_KIND:
            # 4개 이상 같은 수 찾기
            for num, count in counter.most_common():
                if count >= 4:
                    result = [num] * 4
                    others = [d for d in dice_arr if d != num]
                    if others:
                        result.append(max(others))
                    else:
                        result.append(6)
                    return result
            
            # 없으면 가장 많은 수로
            if counter:
                best_num = counter.most_common(1)[0][0]
                result = [best_num] * min(5, counter[best_num])
                while len(result) < 5:
                    others = [d for d in dice_arr if d != best_num]
                    if others:
                        result.append(max(others))
                        if result.count(max(others)) < counter.get(max(others), 0):
                            others.remove(max(others))
                    else:
                        result.append(6)
                return result[:5]
            return [6] * 5
            
        elif rule == DiceRule.FULL_HOUSE:
            counts = counter.most_common()
            
            if len(counts) >= 2:
                three_candidate = None
                two_candidate = None
                
                for num, count in counts:
                    if count >= 3 and three_candidate is None:
                        three_candidate = (num, count)
                    elif count >= 2 and two_candidate is None and num != (three_candidate[0] if three_candidate else None):
                        two_candidate = (num, count)
                        
                if three_candidate and two_candidate:
                    result = [three_candidate[0]] * 3 + [two_candidate[0]] * 2
                    return result
                elif three_candidate and three_candidate[1] >= 5:
                    return [three_candidate[0]] * 5
                elif three_candidate:
                    others = [d for d in dice_arr if d != three_candidate[0]]
                    if others:
                        highest_other = max(others)
                        result = [three_candidate[0]] * 3 + [highest_other] * 2
                        return result
                        
            all_dice = sorted(dice_arr, reverse=True)
            return all_dice[:5]
            
        else:  # CHOICE
            sorted_dice = sorted(dice_arr, reverse=True)
            return sorted_dice[:5]

    def _has_large_straight(self, unique_nums: List[int]) -> bool:
        """라지 스트레이트 확인"""
        if len(unique_nums) < 5:
            return False
        straights = [[1,2,3,4,5], [2,3,4,5,6]]
        unique_set = set(unique_nums)
        return any(all(num in unique_set for num in straight) for straight in straights)
    
    def _has_small_straight(self, unique_nums: List[int]) -> bool:
        """스몰 스트레이트 확인"""
        if len(unique_nums) < 4:
            return False
        straights = [{1,2,3,4}, {2,3,4,5}, {3,4,5,6}]
        unique_set = set(unique_nums)
        return any(straight.issubset(unique_set) for straight in straights)

    def _is_valid_combination(self, target_dice: List[int], available_dice: List[int]) -> bool:
        """조합 유효성 검사"""
        if not target_dice or len(target_dice) != 5:
            return False
            
        target_counter = Counter(target_dice)
        available_counter = Counter(available_dice)
        
        for dice_val, needed_count in target_counter.items():
            if available_counter.get(dice_val, 0) < needed_count:
                return False
        return True

    def _make_safe_combination(self, dice_arr: List[int], rule: DiceRule) -> List[int]:
        """안전한 조합 생성"""
        if len(dice_arr) >= 5:
            return sorted(dice_arr, reverse=True)[:5]
        else:
            result = dice_arr.copy()
            while len(result) < 5:
                result.append(6)
            return result

    def update_get(self, dice_a: List[int], dice_b: List[int], my_bid: Bid, opp_bid: Bid, my_group: str):
        """입찰 결과 업데이트"""
        self.dice_history.append((dice_a.copy(), dice_b.copy()))
        
        if my_group == "A":
            self.my_state.add_dice(dice_a)
            self.opp_state.add_dice(dice_b)
        else:
            self.my_state.add_dice(dice_b)
            self.opp_state.add_dice(dice_a)

        my_bid_ok = my_bid.group == my_group
        self.my_state.bid(my_bid_ok, my_bid.amount)

        opp_group = "B" if my_group == "A" else "A"
        opp_bid_ok = opp_bid.group == opp_group
        self.opp_state.bid(opp_bid_ok, opp_bid.amount)
        
        # 상대방 패턴 기록
        self.opp_bid_amounts.append(opp_bid.amount)
        self.opp_bid_groups.append(opp_bid.group)
        self.my_bid_history.append(my_bid.amount)
        
        # 입찰 충돌 기록
        if my_bid.group == opp_bid.group:
            self.bid_conflicts.append(opp_bid.amount)
            self.bidding_wars_total += 1
            if my_bid.amount > opp_bid.amount:
                self.bidding_wars_won += 1
                
        self.round += 1

    def update_put(self, put: DicePut):
        """내 주사위 배치 업데이트"""
        self.my_state.use_dice(put)

    def update_set(self, put: DicePut):
        """상대방 주사위 배치 업데이트"""
        self.opp_state.use_dice(put)
        if put.rule is not None:
            self.opp_rule_priorities.append(put.rule.value)
            self.opp_puts_history.append((put.rule.value, put.dice.copy()))

class GameState:
    def __init__(self):
        self.dice = []
        self.rule_score: List[Optional[int]] = [None] * 12
        self.bid_score = 0

    def get_upper_section_score(self) -> int:
        """기본 점수 구간 합계"""
        return sum(score for score in self.rule_score[0:6] if score is not None)

    def get_total_score(self) -> int:
        """총점 계산"""
        basic = self.get_upper_section_score()
        bonus = 35000 if basic >= 63000 else 0
        combination = sum(score for score in self.rule_score[6:12] if score is not None)
        return basic + bonus + combination + self.bid_score

    def bid(self, is_successful: bool, amount: int):
        """입찰 점수 적용"""
        if is_successful:
            self.bid_score -= amount
        else:
            self.bid_score += amount

    def add_dice(self, new_dice: List[int]):
        """주사위 추가"""
        self.dice.extend(new_dice)

    def use_dice(self, put: DicePut):
        """주사위 사용"""
        if put.rule is not None and self.rule_score[put.rule.value] is not None:
            return
        
        if not self.dice:
            if put.rule is not None:
                self.rule_score[put.rule.value] = 0
            return
        
        # 사용할 주사위 결정
        if len(self.dice) <= 5:
            use_dice = self.dice.copy()
            while len(use_dice) < 5:
                use_dice.append(1)
        else:
            # 요청된 조합이 가능한지 확인하고 최적화
            available_counter = Counter(self.dice)
            needed_counter = Counter(put.dice[:5])
            
            use_dice = []
            temp_available = available_counter.copy()
            
            # 요청된 주사위를 최대한 정확히 사용
            for dice_val in put.dice[:5]:
                if temp_available.get(dice_val, 0) > 0:
                    use_dice.append(dice_val)
                    temp_available[dice_val] -= 1
                    
            # 부족한 만큼 최적의 대안으로 채우기
            while len(use_dice) < 5:
                available_dice = []
                for val, count in temp_available.items():
                    available_dice.extend([val] * count)
                    
                if available_dice:
                    # 규칙에 따른 최적의 선택
                    if put.rule and put.rule.value < 6:  # 기본 규칙
                        target_val = put.rule.value + 1
                        if target_val in available_dice:
                            best_dice = target_val
                        else:
                            best_dice = max(available_dice)
                    else:
                        best_dice = max(available_dice)
                        
                    use_dice.append(best_dice)
                    temp_available[best_dice] -= 1
                    if temp_available[best_dice] <= 0:
                        del temp_available[best_dice]
                else:
                    use_dice.append(1)
        
        # 정확히 5개 사용
        use_dice = use_dice[:5]
        
        # 보유 주사위에서 제거
        temp_dice = self.dice.copy()
        for dice_val in use_dice:
            if dice_val in temp_dice:
                temp_dice.remove(dice_val)
        self.dice = temp_dice
        
        # 점수 계산
        if put.rule is not None:
            final_put = DicePut(put.rule, use_dice)
            self.rule_score[put.rule.value] = self.calculate_score(final_put)

    @staticmethod
    def calculate_score(put: DicePut) -> int:
        """점수 계산 함수"""
        rule, dice = put.rule, put.dice

        if rule == DiceRule.ONE:
            return sum(d for d in dice if d == 1) * 1000
        if rule == DiceRule.TWO:
            return sum(d for d in dice if d == 2) * 1000
        if rule == DiceRule.THREE:
            return sum(d for d in dice if d == 3) * 1000
        if rule == DiceRule.FOUR:
            return sum(d for d in dice if d == 4) * 1000
        if rule == DiceRule.FIVE:
            return sum(d for d in dice if d == 5) * 1000
        if rule == DiceRule.SIX:
            return sum(d for d in dice if d == 6) * 1000
        if rule == DiceRule.CHOICE:
            return sum(dice) * 1000
        if rule == DiceRule.FOUR_OF_A_KIND:
            ok = any(dice.count(i) >= 4 for i in range(1, 7))
            return sum(dice) * 1000 if ok else 0
        if rule == DiceRule.FULL_HOUSE:
            counter = Counter(dice)
            counts = sorted(counter.values(), reverse=True)
            ok = (len(counts) >= 2 and counts[0] >= 3 and counts[1] >= 2) or counts[0] == 5
            return sum(dice) * 1000 if ok else 0
        if rule == DiceRule.SMALL_STRAIGHT:
            unique = set(dice)
            straights = [{1,2,3,4}, {2,3,4,5}, {3,4,5,6}]
            ok = any(straight.issubset(unique) for straight in straights)
            return 15000 if ok else 0
        if rule == DiceRule.LARGE_STRAIGHT:
            unique = set(dice)
            straights = [{1,2,3,4,5}, {2,3,4,5,6}]
            ok = any(straight.issubset(unique) for straight in straights)
            return 30000 if ok else 0
        if rule == DiceRule.YACHT:
            ok = any(dice.count(i) == 5 for i in range(1, 7))
            return 50000 if ok else 0

        return 0

def main():
    game = Game()
    dice_a, dice_b = [0] * 5, [0] * 5
    my_bid = Bid("", 0)

    while True:
        try:
            line = input().strip()
            if not line:
                continue

            command, *args = line.split()

            if command == "READY":
                print("OK")
                sys.stdout.flush()
                continue

            if command == "ROLL":
                str_a, str_b = args
                for i, c in enumerate(str_a):
                    dice_a[i] = int(c)
                for i, c in enumerate(str_b):
                    dice_b[i] = int(c)
                my_bid = game.calculate_bid_targeted(dice_a, dice_b)
                print(f"BID {my_bid.group} {my_bid.amount}")
                sys.stdout.flush()
                continue

            if command == "GET":
                get_group, opp_group, opp_score = args
                opp_score = int(opp_score)
                opp_bid = Bid(opp_group, opp_score)
                game.update_get(dice_a, dice_b, my_bid, opp_bid, get_group)
                continue

            if command == "SCORE":
                put = game.calculate_put_targeted()
                game.update_put(put)
                if put and put.rule is not None:
                    print(f"PUT {put.rule.name} {''.join(map(str, put.dice))}")
                else:
                    print("PUT CHOICE 66666")
                sys.stdout.flush()
                continue

            if command == "SET":
                rule, str_dice = args
                dice = [int(c) for c in str_dice]
                game.update_set(DicePut(DiceRule[rule], dice))
                continue

            if command == "FINISH":
                break

        except EOFError:
            break
        except Exception as e:
            # 안전 장치
            try:
                if "ROLL" in line:
                    print("BID A 0")
                elif "SCORE" in line:
                    print("PUT CHOICE 66666")
                sys.stdout.flush()
            except:
                pass
            break

if __name__ == "__main__":
    main()
